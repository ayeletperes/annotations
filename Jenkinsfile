pipeline {
  agent { label "${params.AGENT_LABEL}" }
  options { 
    timestamps()
    ansiColor('xterm')
  }

  parameters {
    // Execution Parameters
    string(name: 'TASKS', defaultValue: 'run-A,run-B', description: 'Comma separated run names')
    choice(name: 'EXECUTION_MODE', choices: ['slurm_array', 'jenkins_parallel', 'jenkins_sequential'], 
           description: 'How to submit to SLURM: array (SLURM manages), parallel (Jenkins submits multiple), or sequential (Jenkins submits one-by-one)')
    string(name: 'MAX_CONCURRENCY', defaultValue: '5', description: 'Max concurrent jobs (for array and parallel modes)')
    
    // Nextflow Parameters
    string(name: 'NEXTFLOW_PIPELINE', defaultValue: 'nf-core/airrflow', description: 'Nextflow pipeline to run')
    string(name: 'NEXTFLOW_VERSION', defaultValue: '4.3.1', description: 'Pipeline version/revision')
    string(name: 'NEXTFLOW_PROFILE', defaultValue: 'test,mccleary', description: 'Nextflow profiles (comma-separated)')
    string(name: 'NEXTFLOW_CONFIG', defaultValue: '', description: 'Additional Nextflow config file (optional)')
    text(name: 'NEXTFLOW_PARAMS', defaultValue: '', description: 'Additional Nextflow parameters (one per line, format: --param value)')
    
    // Directory Parameters
    string(name: 'SCRATCH_BASE', defaultValue: '${SCRATCH_DIR}/jenkins-runs', description: 'Base scratch directory (use ${SCRATCH_DIR} for user scratch)')
    string(name: 'OUTPUT_BASE', defaultValue: '${SCRATCH_DIR}/nextflow-outputs', description: 'Base output directory')
    
    // SLURM Parameters (used by ALL execution modes!)
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
          
          echo """
          ========================================
          Execution Mode: ${params.EXECUTION_MODE}
          ========================================
          ALL modes submit to SLURM:
          - slurm_array: Single array job, SLURM manages parallelism
          - jenkins_parallel: Multiple individual SLURM jobs in parallel
          - jenkins_sequential: Individual SLURM jobs one at a time
          ========================================
          """
        }
      }
    }

    stage('Prepare Directories') {
      when { expression { !params.ONLY_MONITOR } }
      steps {
        sh '''#!/usr/bin/env bash
          set -euo pipefail
          echo "Setting up directories..."
          echo "Base working directory: ${BASE_WD}/${RUN_ID}"
          echo "Output directory: ${OUTPUT_DIR}"
          mkdir -p "${BASE_WD}/${RUN_ID}"
          mkdir -p "${OUTPUT_DIR}"
        '''
      }
    }

    stage('Submit Jobs to SLURM') {
      when { expression { !params.ONLY_MONITOR } }
      steps {
        script {
          def names = params.TASKS.split(/\s*,\s*/).findAll { it }
          
          switch(params.EXECUTION_MODE) {
            case 'slurm_array':
              submitSlurmArray(names)
              break
            case 'jenkins_parallel':
              submitJenkinsParallelSlurm(names)
              break
            case 'jenkins_sequential':
              submitJenkinsSequentialSlurm(names)
              break
          }
        }
      }
    }

    stage('Monitor Jobs') {
      when { 
        expression { 
          params.ONLY_MONITOR || params.AUTO_MONITOR
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

// ============================================
// SLURM SCRIPT GENERATOR
// ============================================
def generateSlurmScript(taskName = null, arraySpec = null) {
  def nfParams = params.NEXTFLOW_PARAMS?.trim()?.split('\n')?.findAll{it}?.join(' ') ?: ''
  def accountLine = params.SLURM_ACCOUNT?.trim() ? "#SBATCH --account=${params.SLURM_ACCOUNT}" : ""
  
  def isArray = (arraySpec && arraySpec != "1")
  def arrayLine = arraySpec ? "#SBATCH --array=${arraySpec}" : ""
  
  return """#!/usr/bin/env bash
#SBATCH --job-name=nf-${taskName ?: 'array'}-${env.RUN_ID}
#SBATCH --output=slurm-%A${isArray ? '_%a' : ''}.out
#SBATCH --error=slurm-%A${isArray ? '_%a' : ''}.err
#SBATCH --time=${params.SLURM_TIME}
#SBATCH --cpus-per-task=${params.SLURM_CPUS}
#SBATCH --mem-per-cpu=${params.SLURM_MEM}
#SBATCH --partition=${params.SLURM_PARTITION}
${arrayLine}
${accountLine}

set -euo pipefail

echo "Starting on host: \$(hostname)"
echo "Job ID: \${SLURM_JOB_ID}${isArray ? ', Array Task: \${SLURM_ARRAY_TASK_ID}' : ''}"

# Load modules if specified
${params.JAVA_MODULE?.trim() ? "module load ${params.JAVA_MODULE} || true" : ""}

# Determine task name
${isArray ? '''
# Array job: get task name from file
TASKS_FILE="${SLURM_SUBMIT_DIR}/tasks.tsv"
TASK_NAME=$(sed -n "${SLURM_ARRAY_TASK_ID}p" "${TASKS_FILE}" | tr -d '\\r\\n')
if [ -z "${TASK_NAME}" ]; then
  echo "ERROR: No task found at index ${SLURM_ARRAY_TASK_ID}"
  exit 1
fi
''' : """
# Single job: task name is predefined
TASK_NAME="${taskName}"
"""}

# Setup directories
WORK_DIR="\${SLURM_SUBMIT_DIR}/\${TASK_NAME}"
OUT_DIR="${env.OUTPUT_DIR}/\${TASK_NAME}"
mkdir -p "\${WORK_DIR}" "\${OUT_DIR}"

echo "Running task: \${TASK_NAME}"
echo "Work directory: \${WORK_DIR}"
echo "Output directory: \${OUT_DIR}"

# Run Nextflow
cd "\${WORK_DIR}"
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

EXIT_CODE=\$?
echo "Task \${TASK_NAME} completed with exit code: \${EXIT_CODE}"
exit \${EXIT_CODE}
"""
}

// ============================================
// SLURM ARRAY SUBMISSION
// ============================================
def submitSlurmArray(names) {
  int N = names.size()
  int CONC = params.MAX_CONCURRENCY.toInteger()
  if (CONC > N) CONC = N
  
  String arraySpec = (N > 1) ? "1-${N}%${CONC}" : "1"
  echo "Submitting SLURM array job for ${N} tasks with spec: ${arraySpec}"
  
  // Write tasks file
  writeFile file: "tasks.tsv", text: names.join("\n") + "\n"
  
  // Generate SLURM script using unified function
  def slurmScript = generateSlurmScript(null, arraySpec)
  writeFile file: "array_job.slurm", text: slurmScript
  
  sh '''#!/usr/bin/env bash
    set -euo pipefail
    
    # Copy files to run directory
    RUN_DIR="${BASE_WD}/${RUN_ID}"
    cp tasks.tsv array_job.slurm "${RUN_DIR}/"
    
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
    echo "✓ Submitted SLURM array job: ${JOB_ID}"
  '''
}

// ============================================
// JENKINS PARALLEL SLURM SUBMISSION
// ============================================
def submitJenkinsParallelSlurm(names) {
  def parallelTasks = [:]
  def jobIds = Collections.synchronizedList([])
  
  echo "Submitting ${names.size()} individual SLURM jobs in parallel"
  
  names.each { taskName ->
    parallelTasks[taskName] = {
      stage("Submit: ${taskName}") {
        // Generate script for single task using unified function
        def slurmScript = generateSlurmScript(taskName, null)
        writeFile file: "${taskName}.slurm", text: slurmScript
        
        def jobId = sh(
          script: """#!/usr/bin/env bash
            set -euo pipefail
            RUN_DIR="${env.BASE_WD}/${env.RUN_ID}"
            cp "${taskName}.slurm" "\${RUN_DIR}/"
            cd "\${RUN_DIR}"
            sbatch "${taskName}.slurm" | awk '/Submitted batch job/ {print \$4}'
          """,
          returnStdout: true
        ).trim()
        
        jobIds.add(jobId)
        echo "✓ Submitted SLURM job ${jobId} for ${taskName}"
      }
    }
  }
  
  // Run submissions in parallel
  parallel parallelTasks
  
  // Save all job IDs
  writeFile file: ".job_status", text: "SLURM_INDIVIDUAL:${jobIds.join(',')}:${env.BASE_WD}/${env.RUN_ID}\n"
  echo "All ${jobIds.size()} SLURM jobs submitted: ${jobIds.join(', ')}"
}

// ============================================
// JENKINS SEQUENTIAL SLURM SUBMISSION
// ============================================
def submitJenkinsSequentialSlurm(names) {
  def jobIds = []
  
  echo "Submitting ${names.size()} SLURM jobs sequentially"
  
  names.each { taskName ->
    stage("Submit: ${taskName}") {
      // Generate script for single task using unified function
      def slurmScript = generateSlurmScript(taskName, null)
      writeFile file: "${taskName}.slurm", text: slurmScript
      
      def jobId = sh(
        script: """#!/usr/bin/env bash
          set -euo pipefail
          RUN_DIR="${env.BASE_WD}/${env.RUN_ID}"
          cp "${taskName}.slurm" "\${RUN_DIR}/"
          cd "\${RUN_DIR}"
          sbatch "${taskName}.slurm" | awk '/Submitted batch job/ {print \$4}'
        """,
        returnStdout: true
      ).trim()
      
      jobIds.add(jobId)
      echo "✓ Submitted SLURM job ${jobId} for ${taskName}"
      
      // Wait for completion before submitting next
      if (params.AUTO_MONITOR && names.size() > 1) {
        echo "Waiting for ${taskName} to complete..."
        sh """#!/usr/bin/env bash
          while true; do
            STATE=\$(sacct -j ${jobId} --format=State -n -P | head -1)
            case "\$STATE" in
              COMPLETED)
                echo "✓ Job ${jobId} completed successfully"
                break
                ;;
              FAILED|CANCELLED|TIMEOUT|OUT_OF_MEMORY)
                echo "✗ Job ${jobId} failed with state: \$STATE"
                exit 1
                ;;
              RUNNING)
                echo "  Job ${jobId} is running..."
                sleep 30
                ;;
              PENDING)
                echo "  Job ${jobId} is pending..."
                sleep 10
                ;;
              *)
                sleep 10
                ;;
            esac
          done
        """
      }
    }
  }
  
  // Save all job IDs
  writeFile file: ".job_status", text: "SLURM_INDIVIDUAL:${jobIds.join(',')}:${env.BASE_WD}/${env.RUN_ID}\n"
  echo "All ${jobIds.size()} SLURM jobs submitted sequentially: ${jobIds.join(', ')}"
}

// ============================================
// JOB MONITORING
// ============================================
def monitorJobs() {
  sh '''#!/usr/bin/env bash
    # Be strict on unset vars, but do not die on non-zero pipelines
    set -u

    # Helpers
    have() { command -v "$1" >/dev/null 2>&1; }

    get_job_state() {
      # Args: JOB_ID
      local J="$1" st=""
      # Prefer sacct for terminal states
      if have sacct; then
        # Filter to the parent job or .batch step. Do not fail if sacct errors.
        st=$(sacct -j "$J" --parsable2 --noheader -o JobID,State 2>/dev/null \
              | awk -F'|' -v j="$J" '($1==j || $1 ~ "\\.batch$"){print $2}' \
              | tail -n1 || true)
      fi
      # If sacct gave nothing, try squeue for live state
      if [ -z "${st}" ] && have squeue; then
        st=$(squeue -h -j "$J" -o "%T" 2>/dev/null | head -n1 || true)
      fi
      # Normalize and default
      st=$(echo "${st:-UNKNOWN}" | tr '[:lower:]' '[:upper:]')
      printf "%s" "$st"
    }

    # Get job info
    if [ -f ".job_status" ]; then
      JOB_INFO=$(cat .job_status)
    elif [ -n "${EXISTING_JOB_ID:-}" ]; then
      if [[ "${EXISTING_JOB_ID}" =~ , ]]; then
        JOB_INFO="SLURM_INDIVIDUAL:${EXISTING_JOB_ID}:unknown"
      else
        JOB_INFO="SLURM_ARRAY:${EXISTING_JOB_ID}:unknown"
      fi
    else
      echo "No job information found"
      exit 1
    fi

    JOB_TYPE=$(echo "$JOB_INFO" | cut -d: -f1)
    JOB_IDS=$(echo "$JOB_INFO" | cut -d: -f2)
    RUN_DIR=$(echo "$JOB_INFO" | cut -d: -f3)

    echo "Monitoring ${JOB_TYPE} jobs: ${JOB_IDS}"

    if [ "$JOB_TYPE" = "SLURM_ARRAY" ]; then
      monitor_array_job() {
        local JOB_ID=$1
        declare -A TASK_STATUS
        while true; do
          echo "=== Status check $(date '+%H:%M:%S') ==="
          local TOTAL=0 COMPLETED=0 FAILED=0 RUNNING=0 PENDING=0

          # Use sacct if present, else fall back to squeue listing of array tasks
          if have sacct; then
            MAPFILE -t lines < <(sacct -j "${JOB_ID}" --format=JobID,State -P -n 2>/dev/null || true)
          else
            MAPFILE -t lines < <(squeue -h -j "${JOB_ID}" -o "%i|%T" 2>/dev/null || true)
          fi

          for line in "${lines[@]}"; do
            [ -n "$line" ] || continue
            jobid="${line%%|*}"
            state="${line#*|}"
            if [[ "$jobid" =~ ^${JOB_ID}_[0-9]+$ ]]; then
              TOTAL=$((TOTAL + 1))
              TASK_NUM="${jobid##*_}"
              OLD_STATE="${TASK_STATUS[$TASK_NUM]:-NEW}"
              if [ "$OLD_STATE" != "$state" ]; then
                echo "  Task ${TASK_NUM}: ${OLD_STATE} -> ${state}"
                TASK_STATUS[$TASK_NUM]="$state"
              fi
              case "${state^^}" in
                COMPLETED|COMPLETING) ((COMPLETED++)) ;;
                FAILED|CANCELLED|TIMEOUT|OUT_OF_MEMORY|NODE_FAIL|PREEMPTED|BOOT_FAIL)
                  ((FAILED++)); echo "  Task ${TASK_NUM} failed: ${state}" ;;
                RUNNING) ((RUNNING++)) ;;
                PENDING|CONFIGURING|SUSPENDED) ((PENDING++)) ;;
                *) : ;;
              esac
            fi
          done

          echo "Summary: Total=${TOTAL}, Running=${RUNNING}, Pending=${PENDING}, Completed=${COMPLETED}, Failed=${FAILED}"

          if [ $TOTAL -gt 0 ] && [ $((COMPLETED + FAILED)) -eq $TOTAL ]; then
            echo "All tasks completed."
            [ $FAILED -gt 0 ] && exit 1
            break
          fi
          sleep 30
        done
      }
      monitor_array_job "$JOB_IDS"

    else
      monitor_individual_jobs() {
        IFS=',' read -ra JOB_ARRAY <<< "$1"
        local TOTAL=${#JOB_ARRAY[@]}
        while true; do
          echo "=== Status check $(date '+%H:%M:%S') ==="
          local COMPLETED=0 FAILED=0 RUNNING=0 PENDING=0

          for JOB_ID in "${JOB_ARRAY[@]}"; do
            STATE="$(get_job_state "$JOB_ID")"
            case "$STATE" in
              COMPLETED|COMPLETING) ((COMPLETED++)) ;;
              FAILED|CANCELLED|TIMEOUT|OUT_OF_MEMORY|NODE_FAIL|PREEMPTED|BOOT_FAIL)
                ((FAILED++)); echo "  Job ${JOB_ID} failed: ${STATE}" ;;
              RUNNING) ((RUNNING++)) ;;
              PENDING|CONFIGURING|SUSPENDED) ((PENDING++)) ;;
              UNKNOWN|"") : ;;  # neither sacct nor squeue know yet
            esac
          done

          echo "Summary: Total=${TOTAL}, Running=${RUNNING}, Pending=${PENDING}, Completed=${COMPLETED}, Failed=${FAILED}"

          if [ $((COMPLETED + FAILED)) -eq $TOTAL ]; then
            echo "All jobs completed."
            [ $FAILED -gt 0 ] && exit 1
            break
          fi
          sleep 30
        done
      }
      monitor_individual_jobs "$JOB_IDS"
    fi

    echo
    echo "=== Final Job Summary ==="
    if have sacct; then
      if [ "$JOB_TYPE" = "SLURM_ARRAY" ]; then
        sacct -j "${JOB_IDS}" --format=JobID,JobName,State,ExitCode,Elapsed,MaxRSS -n || true
      else
        IFS=',' read -ra JOB_ARRAY <<< "$JOB_IDS"
        for JOB_ID in "${JOB_ARRAY[@]}"; do
          sacct -j "${JOB_ID}" --format=JobID,JobName,State,ExitCode,Elapsed,MaxRSS -n || true
        done
      fi
    else
      echo "sacct not available. Skipping final sacct summary."
    fi
  '''
}
