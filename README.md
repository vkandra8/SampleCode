# Power Platform CI/CD Pipeline

This repository contains an Azure DevOps pipeline for automated export, validation, and deployment of Power Platform solutions across multiple environments.

## Pipeline Overview

The pipeline consists of three main stages:

1. **Approval**: Manual approval gate before solution export
2. **Export & Check**: Export solutions and run solution checker
3. **Import**: Deploy solutions to target environment

## Features

- **Multi-solution support**: Process multiple solutions (CM10, CM20, CM30, CM40, CM50, CM60) in a single pipeline run
- **Flexible selection**: Choose individual solutions or all solutions via pipeline parameters
- **Environment targeting**: Deploy to development, staging, or production environments
- **Solution validation**: Automated solution checker integration
- **Approval gates**: Manual approval before export and environment-specific deployments

## Pipeline Parameters

### solutionsToProcess
- **Type**: String
- **Default**: `CM10,CM20,CM30,CM40,CM50,CM60`
- **Options**: Individual solutions or comma-separated combinations
- **Description**: Select which solutions to process in the pipeline run

### environmentToDeploy
- **Type**: String
- **Default**: `development`
- **Options**: `development`, `staging`, `production`
- **Description**: Target environment for solution deployment

## Prerequisites

### Variable Groups
Create a variable group named `multiple solutions update` with the following variables:

#### Solution Mappings
For each solution (CM10, CM20, etc.), define:
- `solutionName_CM10`: Actual solution name in Power Platform
- `version_CM10`: Solution version number
- `artifactName_CM10`: Artifact name for build outputs

Example:
```
solutionName_CM10: "Customer Management Core"
version_CM10: "1.0.0.1"
artifactName_CM10: "CM10_Artifacts"
```

#### Service Connections
- `PowerPlatformSPN`: Service connection for source environment
- `PowerPlatformSPN_development`: Service connection for development environment
- `PowerPlatformSPN_staging`: Service connection for staging environment
- `PowerPlatformSPN_production`: Service connection for production environment

### Environments
Create the following environments in Azure DevOps:
- `PreExportApproval`: For manual approval before export
- `development`: Development environment
- `staging`: Staging environment
- `production`: Production environment

## File Structure

```
├── azure-pipelines.yml           # Main pipeline definition
├── templates/
│   ├── export-and-check-template.yml  # Export and validation template
│   └── import-template.yml            # Import template
└── README.md                     # This documentation
```

## Pipeline Stages

### Stage 1: Approval
- **Purpose**: Manual gate for export approval
- **Environment**: PreExportApproval
- **Actions**: Display approval message and wait for manual approval

### Stage 2: Export & Check
- **Purpose**: Export solutions and run validation
- **Dependencies**: Approval stage
- **Actions**:
  - Install Power Platform Tools
  - Export managed and unmanaged solutions
  - Run Solution Checker analysis
  - Publish artifacts for deployment

### Stage 3: Import
- **Purpose**: Deploy solutions to target environment
- **Dependencies**: Export & Check stage
- **Environment**: Selected via parameter
- **Actions**:
  - Download exported artifacts
  - Import managed solutions
  - Publish customizations

## Template Details

### export-and-check-template.yml
Handles individual solution export and validation:
- Exports both managed and unmanaged versions
- Runs Solution Checker with default ruleset
- Publishes artifacts for deployment stage

### import-template.yml
Handles individual solution import:
- Downloads solution artifacts
- Imports managed solution with proper settings
- Publishes customizations

## Usage

1. **Manual Trigger**: Run pipeline manually from Azure DevOps
2. **Select Solutions**: Choose which solutions to process
3. **Select Environment**: Choose target deployment environment
4. **Approve Export**: Manually approve the export stage
5. **Monitor Progress**: Track export, validation, and import stages

## Best Practices

### Solution Management
- Maintain consistent naming conventions for solutions
- Version solutions appropriately
- Test solutions in development before promoting

### Pipeline Security
- Use service principals for authentication
- Implement proper approval gates
- Restrict environment access appropriately

### Monitoring
- Review Solution Checker results regularly
- Monitor deployment logs for issues
- Set up alerts for pipeline failures

## Troubleshooting

### Common Issues

1. **Variable Group Not Found**
   - Ensure the variable group exists and is accessible
   - Check variable naming conventions

2. **Service Connection Errors**
   - Verify service principal permissions
   - Check environment URLs and authentication

3. **Solution Export Failures**
   - Verify solution exists in source environment
   - Check solution dependencies
   - Review export timeouts

4. **Import Failures**
   - Check target environment capacity
   - Verify solution compatibility
   - Review dependency requirements

### Pipeline Logs
Monitor these key areas in pipeline logs:
- Power Platform tool installation
- Solution export/import operations
- Solution Checker results
- Artifact publish/download operations

## Customization

### Adding New Solutions
1. Add solution code to parameter values
2. Add corresponding variables to variable group
3. Test pipeline with new solution

### Environment Configuration
1. Create new environment in Azure DevOps
2. Add corresponding service connection variable
3. Update parameter values if needed

### Custom Validation Rules
- Modify Solution Checker ruleset in export template
- Add custom validation steps as needed

## Support

For issues or questions:
1. Check pipeline logs for error details
2. Verify prerequisites are properly configured
3. Review troubleshooting section
4. Contact your Power Platform administrator