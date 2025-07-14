# Azure DevOps APIM Pipeline - Variable Space Issue Solutions

## Problem Description

The original Azure DevOps pipeline was failing with the error:
```
ERROR: unrecognized arguments: Product
```

This error occurs because the `productToAssociate` variable contains `'Sample Product'` with a space, and when used in PowerShell Azure CLI commands, the space causes the Azure CLI to interpret "Product" as a separate argument.

## Root Cause

The issue stems from how PowerShell handles variable expansion in Azure CLI commands when the variable contains spaces. Even with quotes, the Azure DevOps variable substitution can cause parsing issues.

## Solutions

### Solution 1: Use Product ID Without Spaces (RECOMMENDED)

**File:** `azure-pipelines-alternative1.yml`

**Key Changes:**
- Changed `productToAssociate: 'sample-product'` (no spaces)
- Added `productDisplayName: 'Sample Product'` for display purposes
- Uses PowerShell variables to avoid direct substitution issues

**Advantages:**
- Simplest and most reliable solution
- Follows Azure naming best practices (no spaces in IDs)
- Easy to understand and maintain

**Use this solution if:** You can modify the product name to not have spaces.

### Solution 2: Environment Variables Approach

**File:** `azure-pipelines-alternative2.yml`

**Key Changes:**
- Uses environment variables in the `env:` section
- Keeps original `productToAssociate: 'Sample Product'` with spaces
- PowerShell accesses values via `$env:VARIABLE_NAME`

**Advantages:**
- Preserves original variable names with spaces
- Isolates variable handling from command execution
- More robust against expansion issues

**Use this solution if:** You must keep the product name with spaces and prefer PowerShell.

### Solution 3: Bash Script Approach

**File:** `azure-pipelines-alternative3.yml`

**Key Changes:**
- Uses `scriptType: 'bash'` instead of PowerShell
- Environment variables with `"$VARIABLE_NAME"` syntax
- Better handling of spaces in bash

**Advantages:**
- Bash handles quoted variables more reliably
- Can keep original variable names with spaces
- Cross-platform approach

**Use this solution if:** You prefer bash scripting or have issues with PowerShell variable handling.

## Recommended Approach

**Use Solution 1** (`azure-pipelines-alternative1.yml`) because:

1. **Simplest**: Eliminates the root cause by removing spaces
2. **Best Practice**: Azure resource IDs should not contain spaces
3. **Most Reliable**: No complex escaping or environment variable workarounds needed
4. **Maintainable**: Easier for future developers to understand

## Implementation Steps

1. Use the updated `azure-pipelines.yml` file (which now implements the recommended solution)
2. If you have an existing product in APIM with spaces, you may need to:
   - Create a new product with the new ID (`sample-product`)
   - Migrate any existing subscriptions
   - Delete the old product (if safe to do so)

## Variable Changes Summary

```yaml
# OLD (problematic)
productToAssociate: 'Sample Product'

# NEW (recommended)
productToAssociate: 'sample-product'
productDisplayName: 'Sample Product'
```

## Testing

To test any of these solutions:

1. Commit the chosen pipeline file as `azure-pipelines.yml`
2. Ensure your ARM templates and Function App are properly configured
3. Run the pipeline
4. Verify that the APIM configuration completes without the "unrecognized arguments" error

## Additional Notes

- All solutions maintain the same functionality as the original pipeline
- The error handling and retry logic have been improved in all versions
- Consider using Azure CLI version 2.x latest for best compatibility