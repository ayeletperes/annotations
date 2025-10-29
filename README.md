# Nextflow Jenkins Pipeline

A flexible Jenkins pipeline for running Nextflow workflows with multiple execution strategies (SLURM array, Jenkins parallel, or sequential execution).

## Features

- **Multiple Execution Modes**: Choose between SLURM array jobs, Jenkins parallel execution, or sequential processing
- **Fully Parameterized**: All sensitive information and configurations are parameterized
- **Built-in Monitoring**: Automatic job monitoring with detailed progress tracking
- **Nextflow Integration**: Full support for nf-core pipelines and custom Nextflow workflows
- **Resource Management**: Configurable SLURM resources and Jenkins concurrency limits
- **Comprehensive Reporting**: Automatic collection of Nextflow reports and traces

## Quick Start

1. Add this repository to your Jenkins instance as a Pipeline from SCM
2. Configure your agent labels and execution parameters
3. Run the pipeline with your desired tasks

## Parameters

### Execution Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `TASKS` | `run-A,run-B` | Comma-separated list of task names |
| `EXECUTION_MODE` | `slurm_array` | Execution strategy: `slurm_array`, `jenkins_parallel`, or `jenkins_sequential` |
| `MAX_CONCURRENCY` | `5` | Maximum concurrent jobs (for array and parallel modes) |

### Nextflow Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `NEXTFLOW_PIPELINE` | `nf-core/airrflow` | Nextflow pipeline to run |
| `NEXTFLOW_VERSION` | `4.3.1` | Pipeline version or revision |
| `NEXTFLOW_PROFILE` | `test,docker` | Comma-separated Nextflow profiles |
| `NEXTFLOW_CONFIG` | | Additional Nextflow config file (optional) |
| `NEXTFLOW_PARAMS` | | Additional Nextflow parameters (one per line) |

### Directory Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `SCRATCH_BASE` | `${SCRATCH_DIR}/jenkins-runs` | Base scratch directory for work files |
| `OUTPUT_BASE` | `${SCRATCH_DIR}/nextflow-outputs` | Base directory for Nextflow outputs |

### SLURM Parameters (for array mode)

| Parameter | Default | Description |
|-----------|---------|-------------|
| `SLURM_PARTITION` | `general` | SLURM partition to use |
| `SLURM_TIME` | `12:00:00` | Time limit for jobs |
| `SLURM_CPUS` | `5` | CPUs per task |
| `SLURM_MEM` | `10G` | Memory per CPU |
| `SLURM_ACCOUNT` | | SLURM account (optional) |

### Control Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `AUTO_MONITOR` | `true` | Automatically monitor jobs after submission |
| `ONLY_MONITOR` | `false` | Skip submission, only monitor existing jobs |
| `EXISTING_JOB_ID` | | Job ID to monitor (when ONLY_MONITOR=true) |

### Environment Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `AGENT_LABEL` | `linux` | Jenkins agent label |
| `JAVA_MODULE` | `Java/21` | Java module to load (optional) |

## Usage Examples

### Running with SLURM Array

```groovy
// Run multiple samples using SLURM array
parameters:
  TASKS: "sample1,sample2,sample3,sample4"
  EXECUTION_MODE: "slurm_array"
  MAX_CONCURRENCY: "2"
  NEXTFLOW_PIPELINE: "nf-core/rnaseq"
  NEXTFLOW_VERSION: "3.14.0"
  SLURM_PARTITION: "compute"
  SLURM_TIME: "24:00:00"
```

### Running with Jenkins Parallel

```groovy
// Run tasks in parallel using Jenkins agents
parameters:
  TASKS: "analysis1,analysis2,analysis3"
  EXECUTION_MODE: "jenkins_parallel"
  MAX_CONCURRENCY: "3"
  NEXTFLOW_PROFILE: "docker,test"
```

### Adding Custom Nextflow Parameters

```groovy
parameters:
  NEXTFLOW_PARAMS: |
    --genome GRCh38
    --skip_trimming
    --max_memory 32.GB
    --max_cpus 8
```

### Monitor Only Mode

```groovy
// Monitor a previously submitted job
parameters:
  ONLY_MONITOR: true
  EXISTING_JOB_ID: "123456"
```

## Execution Modes

### SLURM Array Mode
- Submits all tasks as a SLURM array job
- Efficient for HPC clusters
- Automatic resource management by SLURM
- Supports job dependencies and array limits

### Jenkins Parallel Mode
- Runs tasks in parallel using Jenkins executors
- Good for smaller workloads
- Direct integration with Jenkins workspace
- Real-time logs in Jenkins console

### Jenkins Sequential Mode
- Runs tasks one after another
- Useful for resource-limited environments
- Guaranteed execution order
- Simple debugging and troubleshooting

## Directory Structure

```
${SCRATCH_BASE}/
├── ${RUN_ID}/                 # Unique run directory
│   ├── tasks.tsv              # Task list
│   ├── array_job.slurm        # SLURM submission script
│   └── ${TASK_NAME}/          # Individual task directories
│       ├── work/              # Nextflow work directory
│       ├── report.html        # Nextflow execution report
│       ├── trace.txt          # Execution trace
│       └── timeline.html      # Timeline visualization

${OUTPUT_BASE}/
└── ${TASK_NAME}/              # Nextflow output per task
    └── results/               # Pipeline results
```

## Monitoring

The pipeline includes comprehensive monitoring capabilities:

- **Real-time Status Updates**: Track job states (PENDING, RUNNING, COMPLETED, FAILED)
- **Progress Tracking**: Monitor Nextflow process completion
- **Automatic Polling**: Configurable polling intervals
- **State Change Detection**: Alerts on job state transitions
- **Final Summary**: Complete job statistics and resource usage

## Advanced Configuration

### Custom Nextflow Config

Create a custom config file and reference it:

```groovy
parameters:
  NEXTFLOW_CONFIG: "/path/to/custom.config"
```

### Environment Variables

The pipeline automatically resolves:
- `${SCRATCH_DIR}`: User's scratch directory
- `${HOME}`: User's home directory
- `${RUN_ID}`: Unique timestamp for this run

### Module Loading

For HPC systems with environment modules:

```groovy
parameters:
  JAVA_MODULE: "Java/17.0.2"
```

## Troubleshooting

### Common Issues

1. **Jobs not starting**: Check SLURM partition access and resource availability
2. **Module not found**: Verify module names with `module avail`
3. **Permission denied**: Ensure write access to SCRATCH_BASE and OUTPUT_BASE
4. **Monitoring timeout**: Increase polling limits or check job status manually

### Debug Information

Enable debug output by checking:
- SLURM logs: `${SCRATCH_BASE}/${RUN_ID}/slurm-*.out`
- Nextflow logs: `${SCRATCH_BASE}/${RUN_ID}/${TASK_NAME}/.nextflow.log`
- Jenkins console output

## Best Practices

1. **Resource Allocation**: Set appropriate SLURM resources based on pipeline requirements
2. **Concurrency**: Balance MAX_CONCURRENCY with available resources
3. **Output Organization**: Use meaningful task names for easy identification
4. **Monitoring**: Keep AUTO_MONITOR enabled for long-running jobs
5. **Profiles**: Use appropriate Nextflow profiles for your environment

## Contributing

Contributions are welcome! Please ensure:
- All sensitive information is parameterized
- New features include documentation
- Execution modes remain independent
- Error handling is comprehensive

## License

[MIT License](LICENSE)

## Support

For issues and questions:
- Check the troubleshooting section
- Review Jenkins build logs
- Contact your system administrator
- Open an issue on GitHub
