pipeline {
  agent { label "${params.AGENT_LABEL}" }
  options { 
    timestamps()
    ansiColor('xterm')
  }

  parameters {
    // Execution Parameters
    string(name: 'TASKS', defaultValue: 'run-A,run-B', description: 'Comma separated run names')
    choice(name: 'EXECUTION_MODE', choices: ['slurm_array', 'jenkins_parallel', 'jenkins_sequential'], description: 'Execution strategy')
    string(name: 'MAX_CONCURRENCY', defaultValue: '5', description: 'Max concurrent jobs (for array and parallel modes)')
    
    // Nextflow Parameters
    string(name: 'NEXTFLOW_PIPELINE', defaultValue: 'nf-core/airrflow', description: 'Nextflow pipeline to run')
    string(name: 'NEXTFLOW_VERSION', defaultValue: '4.3.1', description: 'Pipeline version/revision')
    string(name: 'NEXTFLOW_PROFILE', defaultValue: 'test,docker', description: 'Nextflow profiles (comma-separated)')
    string(name: 'NEXTFLOW_CONFIG', defaultValue: '', description: 'Additional Nextflow config file (optional)')
    text(name: 'NEXTFLOW_PARAMS', defaultValue: '', description: 'Additional Nextflow parameters (one per line, format: --param value)')
    
    // Directory Parameters
    string(name: 'SCRATCH_BASE', defaultValue: '${SCRATCH_DIR}/jenkins-runs', description: 'Base scratch directory (use ${SCRATCH_DIR} for user scratch)')
    string(name: 'OUTPUT_BASE', defaultValue: '${SCRATCH_DIR}/nextflow-outputs', description: 'Base output directory')
    
    // SLURM Parameters
    string(name: 'SLURM_PARTITION', defaultValue: 'general', description: 'SLURM partition')
    string(name: 'SLURM_TIME', defaultValue: '12:00:00', description: 'SLURM time limit')
    string(name: 'SLURM_CPUS', defaultValue: '5', description: 'CPUs per task')
    string(name: 'SLURM_MEM', defaultValue: '10G', description: 'Memory per CPU')
    string(name: 'SLURM_ACCOUNT', defaultValue: '', description: 'SLURM account (optional)')
    
    // Control Parameters
    booleanParam(name: 'AUTO_MONITOR', defaultValue: true, description: 'Automatically monitor jobs after submission')
    booleanParam(name: 'ONLY_MONITOR', defaultValue: false, description: 'Skip submission and only monitor existing jobs')
    string(name: 'EXISTING_JOB_ID', defaultValue: '', description: 'Existing job ID(s) when ONLY_MONITOR=true')
    
    // Environment Parameters
    string(name: 'AGENT_LABEL', defaultValue: 'linux', description: 'Jenkins agent label')
    string(name: 'JAVA_MODULE', defaultValue: 'Java/21', description: 'Java module to load (optional)')
  }

  environment {
    // Resolve directory paths
    SCRATCH_DIR = sh(script: 'echo ${SCRATCH_DIR:-${HOME}/scratch}', returnStdout: true).trim()
    BASE_WD = sh(script: "echo ${params.SCRATCH_BASE}", returnStdout: true).trim()
    OUTPUT_DIR = sh(script: "echo ${params.OUTPUT_BASE}", returnStdout: true).trim()
    RUN_ID = sh(script: 'date +%Y%m%d_%H%M%S', returnStdout: true).trim()
  }

  stages {
    stage('Validate Parameters') {
      steps {
        script {
          if (!params.TASKS?.trim() && !params.ONLY_MONITOR) {
            error "No TASKS supplied and not in monitor-only mode"
          }
          if (params.ONLY_MONITOR && !params.EXISTING_JOB_ID?.trim()) {
            error "ONLY_MONITOR requires EXISTING_JOB_ID"
          }
        }
      }
    }

    stage('Prepare Directories') {
      when { expression { !params.ONLY_MONITOR } }
      steps {
        sh '''#!/usr/bin/env bash
          set -euo pipefail
          echo "Setting up directories..."
          echo "Base working directory: ${BASE_WD}"
          echo "Output directory: ${OUTPUT_DIR}"
          mkdir -p "${BASE_WD}/${RUN_ID}"
          mkdir -p "${OUTPUT_DIR}"
        '''
      }
    }

    stage('Submit Jobs') {
      when { expression { !params.ONLY_MONITOR } }
      parallel {
        stage('SLURM Array Submission') {
          when { expression { params.EXECUTION_MODE == 'slurm_array' } }
          steps {
            script {
              submitSlurmArray()
            }
          }
        }
        
        stage('Jenkins Parallel Submission') {
          when { expression { params.EXECUTION_MODE == 'jenkins_parallel' } }
          steps {
            script {
              runJenkinsParallel()
            }
          }
        }
        
        stage('Jenkins Sequential Submission') {
          when { expression { params.EXECUTION_MODE == 'jenkins_sequential' } }
          steps {
            script {
              runJenkinsSequential()
            }
          }
        }
      }
    }

    stage('Monitor Jobs') {
      when { 
        expression { 
          params.ONLY_MONITOR || (params.AUTO_MONITOR && params.EXECUTION_MODE == 'slurm_array')
        } 
      }
      steps {
        script {
          monitorJobs()
        }
      }
    }

    stage('Collect Results') {
      when { expression { !params.ONLY_MONITOR } }
      steps {
        sh '''#!/usr/bin/env bash
          set -euo pipefail
          echo "=== Job Execution Summary ==="
          if [ -f "${WORKSPACE}/.job_status" ]; then
            cat "${WORKSPACE}/.job_status"
          fi
          
          echo ""
          echo "=== Nextflow Reports ==="
          find "${BASE_WD}/${RUN_ID}" -name "*.html" -o -name "*.txt" -o -name "trace-*" 2>/dev/null | head -20 || true
        '''
        
        archiveArtifacts artifacts: '.job_status', allowEmptyArchive: true
      }
    }
  }

  post {
    success {
      echo "Pipeline completed successfully!"
    }
    failure {
      echo "Pipeline failed. Check logs for details."
    }
    always {
      cleanWs(deleteDirs: false, patterns: [[pattern: '.job*', type: 'INCLUDE']])
    }
  }
}

// Function to submit SLURM array
def submitSlurmArray() {
  def names = params.TASKS.split(/\s*,\s*/).findAll { it }
  int N = names.size()
  int CONC = params.MAX_CONCURRENCY.toInteger()
  if (CONC > N) CONC = N
  
  String arraySpec = (N > 1) ? "1-${N}%${CONC}" : "1"
  echo "Submitting SLURM array with spec: ${arraySpec}"
  
  // Write tasks file
  writeFile file: "tasks.tsv", text: names.join("\n") + "\n"
  
  // Build SLURM script
  def slurmScript = generateSlurmScript(arraySpec)
  writeFile file: "array_job.slurm", text: slurmScript
  
  sh '''#!/usr/bin/env bash
    set -euo pipefail
    
    # Copy files to run directory
    RUN_DIR="${BASE_WD}/${RUN_ID}"
    cp tasks.tsv "${RUN_DIR}/"
    cp array_job.slurm "${RUN_DIR}/"
    
    # Submit from run directory
    cd "${RUN_DIR}"
    SUBMIT_OUT=$(sbatch array_job.slurm)
    echo "$SUBMIT_OUT"
    
    # Extract job ID
    JOB_ID=$(echo "$SUBMIT_OUT" | awk '/Submitted batch job/ {print $4}')
    if [ -z "$JOB_ID" ]; then
      echo "Failed to parse job ID from: $SUBMIT_OUT"
      exit 1
    fi
    
    # Save job info
    cd -
    echo "SLURM_ARRAY:${JOB_ID}:${RUN_DIR}" > .job_status
    echo "Submitted SLURM array job: ${JOB_ID}"
  '''
}

def generateSlurmScript(arraySpec) {
  def nfParams = params.NEXTFLOW_PARAMS?.trim()?.split('\n')?.findAll{it}?.join(' ') ?: ''
  def accountLine = params.SLURM_ACCOUNT?.trim() ? "#SBATCH --account=${params.SLURM_ACCOUNT}" : ""
  
  return """#!/usr/bin/env bash
#SBATCH --job-name=nf-array-${env.RUN_ID}
#SBATCH --output=slurm-%A_%a.out
#SBATCH --error=slurm-%A_%a.err
#SBATCH --time=${params.SLURM_TIME}
#SBATCH --cpus-per-task=${params.SLURM_CPUS}
#SBATCH --mem-per-cpu=${params.SLURM_MEM}
#SBATCH --partition=${params.SLURM_PARTITION}
#SBATCH --array=${arraySpec}
${accountLine}

set -euo pipefail

# Load modules if specified
${params.JAVA_MODULE?.trim() ? "module load ${params.JAVA_MODULE} || true" : ""}

# Get task name
TASKS_FILE="\${SLURM_SUBMIT_DIR}/tasks.tsv"
TASK_NAME=\$(sed -n "\${SLURM_ARRAY_TASK_ID}p" "\${TASKS_FILE}" | tr -d '\\r\\n')

if [ -z "\${TASK_NAME}" ]; then
  echo "ERROR: No task found at index \${SLURM_ARRAY_TASK_ID}"
  exit 1
fi

# Setup directories
WORK_DIR="\${SLURM_SUBMIT_DIR}/\${TASK_NAME}"
OUT_DIR="${env.OUTPUT_DIR}/\${TASK_NAME}"
mkdir -p "\${WORK_DIR}" "\${OUT_DIR}"

echo "[\${SLURM_JOB_ID}.\${SLURM_ARRAY_TASK_ID}] Running: \${TASK_NAME}"
echo "Work directory: \${WORK_DIR}"
echo "Output directory: \${OUT_DIR}"

# Run Nextflow
nextflow run ${params.NEXTFLOW_PIPELINE} \\
  -r ${params.NEXTFLOW_VERSION} \\
  -profile ${params.NEXTFLOW_PROFILE} \\
  ${params.NEXTFLOW_CONFIG?.trim() ? "-c ${params.NEXTFLOW_CONFIG}" : ""} \\
  --outdir "\${OUT_DIR}" \\
  -work-dir "\${WORK_DIR}" \\
  -with-report "\${WORK_DIR}/report.html" \\
  -with-trace "\${WORK_DIR}/trace.txt" \\
  -with-timeline "\${WORK_DIR}/timeline.html" \\
  ${nfParams}

echo "Task \${TASK_NAME} completed with exit code: \$?"
"""
}

def runJenkinsParallel() {
  def names = params.TASKS.split(/\s*,\s*/).findAll { it }
  def parallelTasks = [:]
  
  names.each { taskName ->
    parallelTasks[taskName] = {
      stage("Run ${taskName}") {
        runNextflowTask(taskName)
      }
    }
  }
  
  // Limit concurrency
  def maxConcurrent = params.MAX_CONCURRENCY.toInteger()
  if (parallelTasks.size() > maxConcurrent) {
    echo "Running ${parallelTasks.size()} tasks with max concurrency of ${maxConcurrent}"
  }
  
  parallel parallelTasks
}

def runJenkinsSequential() {
  def names = params.TASKS.split(/\s*,\s*/).findAll { it }
  
  names.each { taskName ->
    stage("Run ${taskName}") {
      runNextflowTask(taskName)
    }
  }
}

def runNextflowTask(taskName) {
  sh """#!/usr/bin/env bash
    set -euo pipefail
    
    # Setup directories
    WORK_DIR="${env.BASE_WD}/${env.RUN_ID}/${taskName}"
    OUT_DIR="${env.OUTPUT_DIR}/${taskName}"
    mkdir -p "\${WORK_DIR}" "\${OUT_DIR}"
    
    echo "Running Nextflow task: ${taskName}"
    
    # Load modules if needed
    ${params.JAVA_MODULE?.trim() ? "module load ${params.JAVA_MODULE} || true" : ""}
    
    # Parse additional parameters
    NF_PARAMS=""
    if [ -n "${params.NEXTFLOW_PARAMS?.trim() ?: ''}" ]; then
      NF_PARAMS="${params.NEXTFLOW_PARAMS.trim().split('\n').join(' ')}"
    fi
    
    # Run Nextflow
    nextflow run ${params.NEXTFLOW_PIPELINE} \\
      -r ${params.NEXTFLOW_VERSION} \\
      -profile ${params.NEXTFLOW_PROFILE} \\
      ${params.NEXTFLOW_CONFIG?.trim() ? "-c ${params.NEXTFLOW_CONFIG}" : ""} \\
      --outdir "\${OUT_DIR}" \\
      -work-dir "\${WORK_DIR}" \\
      -with-report "\${WORK_DIR}/report.html" \\
      -with-trace "\${WORK_DIR}/trace.txt" \\
      -with-timeline "\${WORK_DIR}/timeline.html" \\
      \${NF_PARAMS}
    
    echo "Task ${taskName} completed"
  """
}

def monitorJobs() {
  sh '''#!/usr/bin/env bash
    set -euo pipefail
    
    # Get job info
    if [ -f ".job_status" ]; then
      JOB_INFO=$(cat .job_status)
    elif [ -n "${EXISTING_JOB_ID:-}" ]; then
      JOB_INFO="SLURM_ARRAY:${EXISTING_JOB_ID}:unknown"
    else
      echo "No job information found"
      exit 1
    fi
    
    JOB_TYPE=$(echo "$JOB_INFO" | cut -d: -f1)
    JOB_ID=$(echo "$JOB_INFO" | cut -d: -f2)
    
    if [ "$JOB_TYPE" != "SLURM_ARRAY" ]; then
      echo "Monitoring only supported for SLURM arrays currently"
      exit 0
    fi
    
    echo "Monitoring SLURM array job: ${JOB_ID}"
    
    # Monitor loop
    declare -A TASK_STATUS
    POLL_INTERVAL=30
    MAX_POLLS=720  # 6 hours with 30s interval
    POLL_COUNT=0
    
    while [ $POLL_COUNT -lt $MAX_POLLS ]; do
      echo "=== Poll cycle $(date '+%Y-%m-%d %H:%M:%S') ==="
      
      # Query job status
      JOB_DATA=$(sacct -j "${JOB_ID}" --format=JobID,JobName,State,ExitCode,Elapsed -P -n 2>/dev/null || true)
      
      if [ -z "$JOB_DATA" ]; then
        echo "No job data available yet..."
        sleep $POLL_INTERVAL
        POLL_COUNT=$((POLL_COUNT + 1))
        continue
      fi
      
      TOTAL_TASKS=0
      COMPLETED_TASKS=0
      FAILED_TASKS=0
      RUNNING_TASKS=0
      
      while IFS='|' read -r jobid name state exitcode elapsed; do
        if [[ "$jobid" =~ ^${JOB_ID}_[0-9]+$ ]]; then
          TOTAL_TASKS=$((TOTAL_TASKS + 1))
          TASK_NUM="${jobid##*_}"
          
          # Track state changes
          OLD_STATE="${TASK_STATUS[$TASK_NUM]:-NEW}"
          if [ "$OLD_STATE" != "$state" ]; then
            echo "  Task ${TASK_NUM}: ${OLD_STATE} -> ${state} (elapsed: ${elapsed})"
            TASK_STATUS[$TASK_NUM]="$state"
          fi
          
          case "$state" in
            RUNNING)
              RUNNING_TASKS=$((RUNNING_TASKS + 1))
              ;;
            COMPLETED)
              COMPLETED_TASKS=$((COMPLETED_TASKS + 1))
              ;;
            FAILED|CANCELLED|TIMEOUT|OUT_OF_MEMORY)
              FAILED_TASKS=$((FAILED_TASKS + 1))
              echo "  ⚠️ Task ${TASK_NUM} failed with state: ${state}, exit code: ${exitcode}"
              ;;
          esac
        fi
      done <<< "$JOB_DATA"
      
      echo "Summary: Total=${TOTAL_TASKS}, Running=${RUNNING_TASKS}, Completed=${COMPLETED_TASKS}, Failed=${FAILED_TASKS}"
      
      # Check if all tasks are done
      DONE_TASKS=$((COMPLETED_TASKS + FAILED_TASKS))
      if [ $TOTAL_TASKS -gt 0 ] && [ $DONE_TASKS -eq $TOTAL_TASKS ]; then
        echo "All tasks have completed!"
        
        if [ $FAILED_TASKS -gt 0 ]; then
          echo "WARNING: ${FAILED_TASKS} task(s) failed"
          exit 1
        fi
        break
      fi
      
      sleep $POLL_INTERVAL
      POLL_COUNT=$((POLL_COUNT + 1))
    done
    
    if [ $POLL_COUNT -ge $MAX_POLLS ]; then
      echo "Monitoring timeout reached after $((MAX_POLLS * POLL_INTERVAL / 3600)) hours"
      exit 1
    fi
    
    echo ""
    echo "=== Final Job Summary ==="
    sacct -j "${JOB_ID}" --format=JobID,JobName,State,ExitCode,Elapsed,MaxRSS -n
  '''
}
