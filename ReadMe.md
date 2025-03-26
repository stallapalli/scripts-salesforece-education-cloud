# Data Migration Notebook

This notebook, `data-migration.ipynb`, contains PowerShell scripts for migrating data between Salesforce orgs. The scripts are designed to handle various data objects and ensure data integrity during the migration process.

## Table of Contents

- [Data Migration Notebook](#data-migration-notebook)
  - [Table of Contents](#table-of-contents)
  - [Key Sections](#key-sections)
  - [Dependencies](#dependencies)
  - [Usage](#usage)
  - [Binds](#binds)
  - [Dictionary](#dictionary)
  - [Example: Adding a New Field Value to Data Mapping](#example-adding-a-new-field-value-to-data-mapping)
    - [Example Code](#example-code)
  - [Functions](#functions)
    - [Format-DateTime](#format-datetime)
      - [Parameters](#parameters)
      - [Example Usage](#example-usage)
    - [Get-Result](#get-result)
      - [Parameters](#parameters-1)
      - [Example Usage](#example-usage-1)
    - [Get-DictionaryEntry](#get-dictionaryentry)
      - [Parameters](#parameters-2)
      - [Example Usage](#example-usage-2)
    - [Update-Binds](#update-binds)
      - [Parameters](#parameters-3)
      - [Example Usage](#example-usage-3)
    - [New-ArraySlices](#new-arrayslices)
      - [Parameters](#parameters-4)
      - [Example Usage](#example-usage-4)
  - [Commonly Used SF CLI Commands](#commonly-used-sf-cli-commands)
    - [Query Data](#query-data)
    - [Upsert Data](#upsert-data)
  - [Notes](#notes)
    - [Salesforce Automation](#salesforce-automation)

## Key Sections

1. **Org Setup**: Scripts to set up target and source org variables, create directories, and set default directories.
2. **Data Import**: Scripts to import dictionaries of translated values and bindings for primary keys.
3. **User Mapping**: Scripts to map users by email address between source and target orgs.
4. **Reference Data Migration**: Scripts to migrate reference data objects such as institutions, campuses, programs, terms, and scholarships.
5. **Student Data Migration**: Scripts to migrate student data objects such as contacts, enrollment opportunities, activities, and awards.

## Dependencies

To run the scripts in this notebook, you need to have the following dependencies installed:

1. **Salesforce CLI**: The Salesforce CLI is required to execute Salesforce commands. You can download and install it from [Salesforce CLI](https://developer.salesforce.com/tools/sfdxcli).
2. **Salesforce Permisisons**: Ensure you have the necessary permissions to query and write data in both the source and target orgs. This includes permissions to access the objects and fields being migrated. There's a special permission set called "Data Migration" that can be assigned to the authorized user to give access to write to the CreatedDate and LastModifiedDate.
3. **PowerShell**: Ensure you have PowerShell installed on your system. The scripts are written in PowerShell and require it to run.
4. **Jupyter Notebook**: The notebook is written in Jupyter Notebook format and can be run using Jupyter Notebook or JupyterLab.
5. **File System**: The scripts write data to the file system, so ensure you have the necessary permissions to create and write to the appropriate directories.

## Usage

1. **Set Up**: Ensure the target and source org variables are set correctly. See [Authorizing an Org](https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/sfdx_dev_auth_web_flow.htm) for more information.
2. **Run Scripts**: Execute the PowerShell scripts in the notebook to perform the data migration.
3. **Validation**: Use the `validateOnly` flag to run the scripts in validation mode without inserting records.
4. **Update Binds**: After successful upserts, update the binds with the target IDs.

## Binds

The binds dictionary is used to store the mapping between the source and target IDs for each object being migrated. The dictionary is updated after each successful upsert operation to ensure that the target IDs are correctly mapped to the source IDs.

Binds should be routinely cached into a bindings.json file to ensure that the data is not lost in case of a system failure or interruption during the migration process.

A Bind consists of the following properties:

- `SourceId`: The ID of the record in the source org.
- `ReferenceId`: (Optional) The ID of the record in the target org.
- `CompositeId`: (Optional) A composite ID that combines the source and target IDs.

## Dictionary

The dictionary serves to define translations for values. Each topic is assigned as a key in the dictionary, and the values are stored as key-value pairs. The dictionary is used to map values from the source to the target org during data migration. 

When adding new values to the dictionary, ensure that the key is unique and that the values are correctly mapped to the corresponding keys.

Most expected enumerated translations should already be a part of the included dictionary with the exception of state values.

State values can easily be added by identifying the possible values from the source org and mapping them to the corresponding values in the target org.

An easy way of accomplishing this is as follows:
    
```powershell
$states = sf data query -q "SELECT ..., MailingState FROM Contact" -o $source --json |
ConvertFrom-Json |
Select-Object -ExpandProperty Result |
Select-Object -ExpandProperty Records |
Select-Object -ExpandProperty MailingState -Unique

Write-Host ($states | Format-Table -AutoSize)
```
Once the possible values for states have been identified, they can be added to the "states" topic in the dictionary.

```powershell
$dictionary["states"]["SomeState"] = "SomeOtherState"
```
Commonly seen patterns for states include, but aren't limited to:

1. [State Abbreviation]AB
2. AB[State Abbreviation]
3. [State Name]AB[State Abbreviation]
4. [State Name]ABBR[State Abbreviation]
5. choose-a-province[State Abbreviation]
6. [State Name]choose-a-province[State Abbreviation]
7. [State Abbreviation]choose-a-province[State Abbreviation]

## Example: Adding a New Field Value to Data Mapping

To add a new field value into the data mapping, follow these steps:

1. **Add the field's API name to the `$fields` variable**.
2. **Include the new property that it maps to in the `PSCustomObject` instantiation**.

### Example Code

```powershell
// filepath: c:\Users\Nick\Projects\TheCommunitySolution\DataMigration\data-migration.ipynb
# Fetch the RecordTypeId for the College Accounts
$collegeAccountRT = sf data query -q "SELECT Id FROM RecordType WHERE SObjectType = 'Account' AND DeveloperName = 'College'" -o $source --json |
ConvertFrom-Json |
Select-Object -ExpandProperty Result |
Select-Object -ExpandProperty Records |
Select-Object -ExpandProperty Id

# Add the binds for Accounts
$binds["Account"] = [System.Collections.Generic.List[PSCustomObject]]::new()

$fields = @(
    "Id", 
    "Name", 
    "EnrollmentrxRx__School_Address__c", 
    "EnrollmentrxRx__School_City__c", 
    "EnrollmentrxRx__School_State_Province__c",
    "EnrollmentrxRx__School_Zip__c",
    "CVUE_ID__c",
    "CreatedDate",
    "LastModifiedDate",
    "New_Custom_Field__c"  # Add the new custom field here
)
    
$records = sf data query -q "SELECT $($fields -join ',') FROM EnrollmentrxRx__School__c" -o $source --json | 
ConvertFrom-Json |
Select-Object -ExpandProperty result |
Select-Object -ExpandProperty records |
ForEach-Object {
    $binds["Account"].add([PSCustomObject]@{
            SourceId    = $_.Id
            ReferenceId = $_.Id
        })

    $billingState = Get-DictionaryEntry -k "states" -v $_.EnrollmentrxRx__School_State_Province__c

    return [PSCustomObject]@{
        Name                = $_.Name
        Type                = "Partner"
        RecordTypeId        = $collegeAccountRT
        BillingStreet       = $_.EnrollmentrxRx__School_Address__c
        BillingCity         = $_.EnrollmentrxRx__School_City__c
        BillingStateCode    = $billingState
        BillingPostalCode   = $_.EnrollmentrxRx__School_Zip__c
        BillingCountryCode  = if ($billingState) { "US" } else { $null }
        Anthology_ID__c     = $_.CVUE_ID__c
        Legacy_Id__c        = $_.Id
        CreatedDate         = Format-DateTime -d $_.CreatedDate
        LastModifiedDate    = Format-DateTime -d $_.LastModifiedDate
        New_Custom_Field__c = $_.New_Custom_Field__c  # Map the new custom field here, inline translation can be applied if needed
    }
} | Export-Csv -Path data\InstitutionAccounts.csv -NoTypeInformation -Encoding UTF8

Write-Host ($records | ConvertTo-Json -Depth 2)
Remove-Variable -Name collegeAccountRT
```

## Functions

### Format-DateTime

Formats a date string into various date formats.

#### Parameters

- `dateString`: The date string to be formatted. This parameter is mandatory.
- `format`: The format to convert the date string into. Valid values are "ISO8601", "RFC1123", "Unix", and "Custom". The default value is "ISO8601".
- `UTC`: A switch to indicate if the date should be converted to UTC. This parameter is optional.

#### Example Usage

```powershell
Format-DateTime -dateString "9/1/2021 2:12:16 PM" -format "RFC1123"
```

### Get-Result

Converts a JSON string to a PowerShell object and extracts the 'Result' property. Primarily used for working with Salesforce API responses.

#### Parameters

- `json`: The JSON string to be converted. This parameter is mandatory and accepts input from the pipeline.

#### Example Usage

```powershell
'{\"Result\": {\"Name\": \"John\", \"Age\": 30}}' | Get-Result
```

### Get-DictionaryEntry

Retrieves a value from a dictionary based on a specified key and value. Gracefully handles finding a key from the hashtable with a potentially null value.

#### Parameters

- `key`: The key to look up in the dictionary. This parameter is mandatory.
- `value`: The value to look up in the dictionary. This parameter is mandatory but allows empty strings.

#### Example Usage

```powershell
$dictionary = @{ "Name" = @{ "First" = "John"; "Last" = "Doe" } }
Get-DictionaryEntry -key "Name" -value "First"
```

### Update-Binds

Updates the binds dictionary with data from a specified JSON file.

#### Parameters

- `key`: The key to look up in the binds dictionary. This parameter is mandatory.
- `filePath`: The path to the JSON file containing the data to update the binds dictionary. This parameter is mandatory.
- `identifier`: The identifier used to group the binds dictionary entries. The default value is "SourceId". This parameter is optional.

#### Example Usage

```powershell
Update-Binds -key "exampleKey" -filePath "C:\\path\\to\\file.json"
```

### New-ArraySlices

Splits an array into smaller arrays (slices) of a specified size.

#### Parameters

- `Item`: The array of items to be split into slices. This parameter is mandatory and accepts input from the pipeline.
- `Size`: The size of each slice. The default value is 10.

#### Example Usage

```powershell
1..25 | New-ArraySlices -Size 5
```

## Commonly Used SF CLI Commands

### Query Data
[sf data query](https://developer.salesforce.com/docs/atlas.en-us.sfdx_cli_reference.meta/sfdx_cli_reference/cli_reference_data_commands_unified.htm#cli_reference_data_query_unified)

```powershell
sf data query -query "SELECT Id, Name FROM Account" --target-org $source --json
```
### Upsert Data

[sf data upsert](https://developer.salesforce.com/docs/atlas.en-us.sfdx_cli_reference.meta/sfdx_cli_reference/cli_reference_data_commands_unified.htm#cli_reference_data_upsert_bulk_unified)
```powershell
sf data upsert bulk --file data\InstitutionAccounts.csv -target-org $target --sobject Account --external-id "Name" --json --wait 10
```

## Notes

- The notebook assumes that certain variables and dictionaries are already defined in the scope.
- The scripts include error handling to ensure data integrity and provide meaningful error messages.
- The `Update-Binds` function is used extensively to update the binds dictionary with target IDs after successful upserts.
- CreatedDate and LastModifiedDate SObject fields can only be set on record creation. Subsequent updates to existing records should not include these columns, as they will result in an error being thrown by the Salesforce system.

### Salesforce Automation
Certain Salesforce automations, such as Flows or Declarative Lookup Rollup Summaries, may be triggered by the data migration process. 
It is important to ensure that the appropriate automations are disabled during the migration process to prevent unintended side effects, row lock issues, or increased processing time.