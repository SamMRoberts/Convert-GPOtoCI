# Convert-GPOtoCI
I have been asked by customers if it is possible to use ConfigMgr to monitor compliance of Group Policy settings.  The short answer has always been yes but it is not easy.  If you had to manually identify all the registry keys associated with a GPO and create the individual CI rules this process could takes hours if not days for larger policies.  This script simplifies the process and queries for the values and creates the CIs in a matter of seconds.  I have seen similar scripts however many of the others still require user to create a .POL file for the script to ingest then import a .CAB file into ConfigMgr.  This script fully automates the process.

## What Does it Do?

This script uses both the Configuration Manager and Active Directory PowerShell modules to query for registry keys associated with Group Policies then create the Configuration Items for each of the registry values.  I have updated the script to include a few options on how this gets done.

First, the script will either query for a single group policy or utilize a Resultant Set of Policy to determine applicable policies for a specified system.  To enable the Resultant Set of Policy (RSOP) option include the 'ResultantSetOfPolicy' parameter and specify the system to run RSOP against using the 'ComputerName' parameter.  Leveraging the Active Directory PowerShell module and the Get-GPResultantSetOfPolicy command the script will run an RSOP on a specified system and export the results to a temporary file.  The file is then searched to determine what group policies are applied to the system and what the link order is for those policies.  This will be used to determine what order the group policy data is queried to simulate their link order.  If the 'ResultantSetOfPolicy' option is not used this step is skipped and the script will query only for the one group policy specified.

To get a list of registry keys associated with the group policies the Get-GPRegistryValue command is used to query for the specified group policies.  The full key path, key name, value and data type are all stored in an array.  When the 'ResultantSetOfPolicy' option is used the group policies will be queried starting with the policy that would be applied last.  As each additional policy is queried the script will check to see if that registry key has already been stored with another value.  If the registry key is already present, the additional occurrence of that key will be skipped since the policy that is last applied would over write the lower policies.  If the 'ResultantSetOfPolicy' option is not used the script will only query for the one policy specified.

Once the script has successfully queried for all the associated registry keys and values it utilized the Configuration Manager PowerShell module to create the Configuration Item definition files in xml format.  This will include a setting and rule for each of the registry keys of supported data type (binary values are not supported by DCM).  You can specify the severity of non-compliant settings as well as remediation of non-compliant items using command line parameters.

The final step is to import the Configuration Item definition file into Configuration Manager.  By default, this is done automatically, creating a CI with settings and rules for all associated registry values.  If you do not wish to have this automatically created you can use the 'ExportOnly' parameter which will save the data to a .cab file which can later be manually imported into Configuration Manager.

In my tests, this script can query for and create a CI with over 200 registry values in under 20 seconds.

## How to Use the Script
This script must be executed from a system that has access to both the GroupPolicy and ConfigurationManager PowerShell modules.  The GroupPolicy module is installed with the Remote Admin Tools and the ConfigurationManager module is installed with the ConfigMgr Admin Console.  Additionally, if the 'ResultantSetOfPolicy' option is used the user must have remote admin access to that system.  Extract the .ZIP file and execute the PowerShell script via a PS console.

### Parameters:
*GroupPolicy** *[optional]* - This is enabled by default and will make the script query only for one specified group policy.
**GpoTarget** *[required unless ResultantSetOfPolicy option is used]* - Name of group policy object
**ResultantSetOfPolicy** *[optional]* - Utilizes a resultant set of policy to determine the set of applied GPOs.  Cannot be used in conjunction with the GroupPolicy option.
**ComputerName** *[required when ResultantSetOfPolicy is used]* - Name of system to run RSOP on.
**DomainTarget** *[required]* - Fully qualified domain name
**SiteCode** *[required]* - ConfigMgr site code
**Remediate** *[optional]* - Enable configuration item to remediate non-compliant settings
**Severity** *[optional]* - Sets the severity of non-compliant items.  (None, Informational, Warning or Critical)
**ExportOnly** *[optional]* - Exports the Configuration Item to a CAB file to be manually imported
**Log** *[optional]* - Writes all discovered registry keys and their related GPO name to a file.


#### Example 1:
.\Convert-GPOtoCI.ps1 -GpoTarget "Windows 10 Settings" -DomainTarget contoso.com -SiteCode T01

#### Example 2:
.\Convert-GPOtoCI.ps1 -GpoTarget "Windows 10 Settings" -DomainTarget contoso.com -SiteCode T01 -Remediate

#### Example 3:
.\Convert-GPOtoCI.ps1 -GroupPolicy -GpoTarget "Windows 10 Settings" -DomainTarget contoso.com -SiteCode T01 -Severity Warning

#### Example 4:
.\Convert-GPOtoCI.ps1 -ResultantSetOfPolicy -ComputerName lab-srv-test01 -DomainTarget contoso.com -SiteCode T01

#### Example 5:
.\Convert-GPOtoCI.ps1 -GpoTarget "Windows 10 Settings" -DomainTarget contoso.com -SiteCode T01 -ExportOnly

### Tested On:
Configuration Manager vNext 1706

### Change Log

#### v 1.1.1 (7/12/2017)
Added -ExportOnly switch that will export the Configuration Item data to a .CAB file instead of automatically creating the CIs.  This file can be used to import the CI into ConfigMgr.

#### v 1.2.1 (7/17/2017)
Added -ResultantSetOfPolicy parameter to enable to script to run RSOP against a system to determine the applied group policies then query for all registry keys associated with the applicable policies and settings.

#### v 1.2.3 (9/18/2017)
Added the -Log switch that will log all discovered registry keys and their related Group Policy object to a file named gpo_registry_discovery_mmddyyyy.log in the scripts root directory.

#### v 1.2.4 (9/18/2017)
Fixed bug where registry values were always being logged to file even when the -Log switch was not set.

#### v 1.2.6 (11/6/2017)
Bug fixes.
Allow for creation of User Policy based CIs. 
