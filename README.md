# AIRR-seq Annotations Pipeline

Testing Jenkins pipelines for annotating AIRR-seq (Adaptive Immune Receptor Repertoire sequencing) on the cluster.

## Project Structure

- **`Jenkinsfile`** - Jenkins pipeline script for automated AIRR-seq annotation workflow
- **`JENKINS_SETUP.md`** - Comprehensive guide for setting up Jenkins and GitHub webhooks

## Quick Start

1. **Set up Jenkins**: Follow the instructions in [`JENKINS_SETUP.md`](JENKINS_SETUP.md)
2. **Configure webhook**: Set up GitHub webhook to trigger builds automatically
3. **Customize pipeline**: Update the `Jenkinsfile` with your specific annotation tools and workflows

## Features

- ✅ Automated pipeline triggered by GitHub webhooks
- ✅ Multiple stages: checkout, environment setup, testing, annotation, reporting
- ✅ Configurable parameters for flexible execution
- ✅ Support for various annotation tools (IgBLAST, IMGT, etc.)
- ✅ Artifact archiving and result management
- ✅ Post-build notifications and reporting

## Documentation

- [JENKINS_SETUP.md](JENKINS_SETUP.md) - Complete setup and configuration guide
- [Jenkinsfile](Jenkinsfile) - Pipeline script with detailed comments

## Contributing

Feel free to customize the pipeline according to your specific AIRR-seq annotation workflow requirements.
