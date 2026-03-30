
ok lets start with repo path : https://github.com/wc-platform-engineering/od-preprovisioning-infra no in this repo we have all the foiles of the project for infra all been worked on i removed all the necessary or required stuff need to be changed . go through them again and see if we need to do more stuff >>>> first let start with the ci folder part : https://github.com/wc-platform-engineering/od-preprovisioning-infra/blob/main/.ci/azure/modules/rg-resources/main.tf >>>> // MSI resources resource "azurerm_user_assigned_identity" "app_msi" { name = replace(var.resource_group.name, "rg-", "id-") location = var.resource_group.location resource_group_name = var.resource_group.name tags = var.resource_group.tags } resource "azurerm_role_assignment" "msi_operator" { scope = var.resource_group.id role_definition_name = "Managed Identity Operator" principal_id = var.service_principal_id skip_service_principal_aad_check = true } // Key Vault resources module "key_vault_name" { source = "gitlab.whitecase.com/tf-modules/azure-resourcename/azure" version = "1.0.1" name = var.app_name resource_type = "azurerm_key_vault" suffixes = [var.environment_type, var.resource_group.location] random = false } resource "azurerm_key_vault" "sharepoint" { name = module.key_vault_name.result location = var.resource_group.location resource_group_name = var.resource_group.name enabled_for_disk_encryption = true tenant_id = var.tenant_id soft_delete_retention_days = 7 purge_protection_enabled = true sku_name = "standard" tags = var.resource_group.tags } resource "azurerm_key_vault_secret" "sharepoint" { for_each = var.key_vault_placeholder_secret_names name = each.value value = "placeholder" key_vault_id = azurerm_key_vault.sharepoint.id lifecycle { ignore_changes = [ value, ] } depends_on = [ azurerm_key_vault_access_policy.deployer ] } resource "azurerm_key_vault_access_policy" "deployer" { key_vault_id = azurerm_key_vault.sharepoint.id tenant_id = var.tenant_id object_id = var.deployer_object_id secret_permissions = ["Get", "List", "Set", "Delete"] } resource "azurerm_key_vault_access_policy" "app_msi" { key_vault_id = azurerm_key_vault.sharepoint.id tenant_id = var.tenant_id object_id = azurerm_user_assigned_identity.app_msi.principal_id key_permissions = ["Get", "List", "Decrypt"] secret_permissions = ["Get", "List"] storage_permissions = ["Get", "List"] certificate_permissions = ["Get", "List"] } resource "azurerm_key_vault_access_policy" "admins" { for_each = toset(var.key_vault_admins) key_vault_id = azurerm_key_vault.sharepoint.id tenant_id = var.tenant_id object_id = each.key key_permissions = ["Backup", "Create", "Decrypt", "Delete", "Encrypt", "Get", "Import", "List", "Purge", "Recover", "Restore", "Sign", "UnwrapKey", "Update", "Verify", "WrapKey", "Release", "Rotate", "GetRotationPolicy", "SetRotationPolicy"] secret_permissions = ["Backup", "Delete", "Get", "List", "Purge", "Recover", "Restore", "Set"] storage_permissions = ["Backup", "Delete", "DeleteSAS", "Get", "GetSAS", "List", "ListSAS", "Purge", "Recover", "RegenerateKey", "Restore", "Set", "SetSAS", "Update"] certificate_permissions = ["Backup", "Create", "Delete", "DeleteIssuers", "Get", "GetIssuers", "Import", "List", "ListIssuers", "ManageContacts", "ManageIssuers", "Purge", "Recover", "Restore", "SetIssuers", "Update"] } --------------------------------------- https://github.com/wc-platform-engineering/od-preprovisioning-infra/blob/main/.ci/azure/modules/rg-resources/outputs.tf >>>> output "msi" { value = azurerm_user_assigned_identity.app_msi } output "sharepoint_secret_id" { value = "@Microsoft.KeyVault(VaultName=${azurerm_key_vault.sharepoint.name};SecretName=${azurerm_key_vault_secret.sharepoint.name})" } ---------------------------------------------------------------------------------------- variable "app_name" { type = string } variable "environment_type" { type = string } variable "resource_group" { type = any } variable "service_principal_id" { type = string } variable "tenant_id" { type = string } variable "key_vault_admins" { type = list(string) } variable "deployer_object_id" { type = string } variable "key_vault_placeholder_secret_names" { type = map(string) description = "List of secrets to pre-create in the key vault" default = { APP_TENANT_NAME = "TenantName" APP_SHAREPOINT_USER = "SharepointUser" APP_SHAREPOINT_SECRET = "SharepointSecret" } } ---------------------------------------------------------------------------------- https://github.com/wc-platform-engineering/od-preprovisioning-infra/blob/main/.ci/azure/vars/dev.tfvars >>>> /* Development Environment Terraform Variables */ app_name = "onedrivepreprov" environment_type = "dev" regions = ["eastus"] tags = { "environment_type" : "dev", "owner" : "appdev", "iregion" : "am5" } key_vault_admins = [ "683aac76-b0cd-4ded-9308-8b0548e89567" // andy.barrionuevo.adm3@3law2law.com ] eventhub_resource_group = "rg-aadevents-dev-eastus" eventhub_namespace = "ehn-aadevents-dev-eastus-15ccc" eventhub_name = "evh-aadevents-dev-eastus-aad" ---------------------------------------------------------------------------------------- https://github.com/wc-platform-engineering/od-preprovisioning-infra/blob/main/.ci/azure/vars/prod.tfvars >>>>> /* Production Environment Terraform Variables */ app_name = "onedrivepreprov" environment_type = "prod" regions = ["eastus"] tags = { "environment_type" : "prod", "owner" : "appdev", "iregion" : "am5" } key_vault_admins = [ "270d55cc-d3d6-4463-bd31-02f2dfac755e" // andy.barrionuevo.adm3@whitecase.com ] eventhub_resource_group = "rg-aadevents-prod-eastus" eventhub_namespace = "ehn-aadevents-prod-eastus-64e10" eventhub_name = "evh-aadevents-prod-eastus-aad" ---------------------------------------------------------------------------------------- https://github.com/wc-platform-engineering/od-preprovisioning-infra/blob/main/.ci/azure/.terraform.lock.hcl >>>>> # This file is maintained automatically by "terraform init". # Manual edits may be lost in future updates. provider "registry.terraform.io/hashicorp/azuread" { version = "2.30.0" constraints = ">= 2.29.0" hashes = [ "h1:xzNKb+lWPsBTxJiaAJ8ECZnY+D6QNM9tA1qpEncIba0=", "zh:1c3e89cf19118fc07d7b04257251fc9897e722c16e0a0df7b07fcd261f8c12e7", "zh:2e62c193030e04ebb10cc0526119cf69824bf2d7e4ea5a2f45bd5d5fb7221d36", "zh:2f3c7a35257332d68b778cefc5201a5f044e4914dd03794a4da662ddfe756483", "zh:35d0d3a1b58fdb8b8c4462d6b7e7016042da43ea9cc734ce897f52a73407d9b0", "zh:47ede0cd0206ec953d40bf4a80aa6e59af64e26cbbd877614ac424533dbb693b", "zh:48c190307d4d42ea67c9b8cc544025024753f46cef6ea64db84735e7055a72da", "zh:6fff9b2c6a962252a70a15b400147789ab369b35a781e9d21cce3804b04d29af", "zh:7646980cf3438bff29c91ffedb74458febbb00a996638751fbd204ab1c628c9b", "zh:77aa2fa7ca6d5446afa71d4ff83cb87b70a2f3b72110fc442c339e8e710b2928", "zh:e20b2b2c37175b89dd0db058a096544d448032e28e3b56e2db368343533a9684", "zh:eab175b1dfe9865ad9404dccb6d5542899f8c435095aa7c679314b811c717ce7", "zh:efc862bd78c55d2ff089729e2a34c1831ab4b0644fc11b36ee4ebed00a4797ba", ] } provider "registry.terraform.io/hashicorp/azurerm" { version = "3.32.0" constraints = ">= 3.26.0" hashes = [ "h1:zjaayFSTUOxL0J0vE3MSPGbUNVxbUkQv76THmtrAM4Q=", "zh:3ee1992144e6bf9801c44df0ed1e10413fa83ad605e3ce751cb342dd46904c41", "zh:4f083079909f929b76c0cb2819b107803ecbf26c761832aaa1e7b4a667025665", "zh:52ad565c4bd37c2b4f0bba78639277ef98caaebf2c4c00c67a2659561079c21c", "zh:5ecf7a8470e066cc27b837a8fbc9a02629bb85797007475539983496bcccbc53", "zh:6348154495cd838862b27a9bc0a2714e8f76cd2919df55fce8da0f64ce240ab1", "zh:8325c4f5f65e30bba2537c7df702c80ae29999fba6194c258b075b3cbde5a709", "zh:8b4d33aa76474a9fac9a6859e759c03ffeadb787abf7a9ba5a05b4ca3914c008", "zh:95ccd31450909582ebcf01548ee20df658049783530d79adcb53a601bb163597", "zh:c104f977b96c6402276c82a8d9d6fee14381511e832e9c3593e589e5ee4e708c", "zh:e12372a41a981c24323a467f6c54b0a17e26c85a0fb569e4b733b2a76c9ba6b6", "zh:e80bf9b674914f91ed00984758288b7266ba5772fad728cd1b4cd2f776851ed8", "zh:f569b65999264a9416862bca5cd2a6177d94ccb0424f3a4ef424428912b9cb3c", ] } provider "registry.terraform.io/hashicorp/http" { version = "3.2.1" hashes = [ "h1:DfxMa1zM/0NCFWN5PAxivSHJMNkOAFZvDYQkO72ZQmw=", "zh:088b3b3128034485e11dff8da16e857d316fbefeaaf5bef24cceda34c6980641", "zh:09ed1f2462ea4590b112e048c4af556f0b6eafc7cf2c75bb2ac21cd87ca59377", "zh:39c6b0b4d3f0f65e783c467d3f634e2394820b8aef907fcc24493f21dcf73ca3", "zh:47aab45327daecd33158a36c1a36004180a518bf1620cdd5cfc5e1fe77d5a86f", "zh:4d70a990aa48116ab6f194eef393082c21cf58bece933b63575c63c1d2b66818", "zh:65470c43fda950c7e9ac89417303c470146de984201fff6ef84299ea29e02d30", "zh:78d5eefdd9e494defcb3c68d282b8f96630502cac21d1ea161f53cfe9bb483b3", "zh:842b4dd63e438f5cd5fdfba1c09b8fdf268e8766e6690988ee24e8b25bfd9e8d", "zh:a167a057f7e2d80c78d4b4057538588131fceb983d5c93b07675ad9eb1aa5790", "zh:d0ba69b62b6db788cfe3cf8f7dc6e9a0eabe2927dc119d7fe3fe6573ee559e66", "zh:e28d24c1d5ff24b1d1cc6f0074a1f41a6974f473f4ff7a37e55c7b6dca68308a", "zh:fde8a50554960e5366fd0e1ca330a7c1d24ae6bbb2888137a5c83d83ce14fd18", ] } provider "registry.terraform.io/hashicorp/random" { version = "3.4.3" hashes = [ "h1:xZGZf18JjMS06pFa4NErzANI98qi59SEcBsOcS2P2yQ=", "zh:41c53ba47085d8261590990f8633c8906696fa0a3c4b384ff6a7ecbf84339752", "zh:59d98081c4475f2ad77d881c4412c5129c56214892f490adf11c7e7a5a47de9b", "zh:686ad1ee40b812b9e016317e7f34c0d63ef837e084dea4a1f578f64a6314ad53", "zh:78d5eefdd9e494defcb3c68d282b8f96630502cac21d1ea161f53cfe9bb483b3", "zh:84103eae7251384c0d995f5a257c72b0096605048f757b749b7b62107a5dccb3", "zh:8ee974b110adb78c7cd18aae82b2729e5124d8f115d484215fd5199451053de5", "zh:9dd4561e3c847e45de603f17fa0c01ae14cae8c4b7b4e6423c9ef3904b308dda", "zh:bb07bb3c2c0296beba0beec629ebc6474c70732387477a65966483b5efabdbc6", "zh:e891339e96c9e5a888727b45b2e1bb3fcbdfe0fd7c5b4396e4695459b38c8cb1", "zh:ea4739860c24dfeaac6c100b2a2e357106a89d18751f7693f3c31ecf6a996f8d", "zh:f0c76ac303fd0ab59146c39bc121c5d7d86f878e9a69294e29444d4c653786f8", "zh:f143a9a5af42b38fed328a161279906759ff39ac428ebcfe55606e05e1518b93", ] } ------------------------------------------ https://github.com/wc-platform-engineering/od-preprovisioning-infra/blob/main/.ci/azure/main.tf >>>>>>> terraform { backend "http" { } } provider "azurerm" { features {} } provider "azuread" { } data "azurerm_client_config" "current" {} // Primary Project Setup =============================================================== // - App Resource Group // - Deployment Service Principal / Role Assignment to App Resource Group module "azure_projectinit" { source = "gitlab.whitecase.com/tf-modules/azure-projectinit/azure" version = "0.0.3" app_name = var.app_name environment_type = var.environment_type regions = var.regions tags = var.tags } // Cross-Deployment Resources (Additional resources required ON remote resources) ====== resource "azurerm_eventhub_consumer_group" "app_access" { name = var.app_name namespace_name = data.azurerm_eventhub_namespace.azureadevents.name eventhub_name = data.azurerm_eventhub.auditevents.name resource_group_name = data.azurerm_resource_group.events.name user_metadata = "${var.app_name}-access" } resource "azurerm_eventhub_consumer_group" "onedrive_storage_access" { name = "${var.app_name}-onedrive" namespace_name = data.azurerm_eventhub_namespace.azureadevents.name eventhub_name = data.azurerm_eventhub.auditevents.name resource_group_name = data.azurerm_resource_group.events.name user_metadata = "${var.app_name}-access" } resource "azurerm_eventhub_authorization_rule" "app_access" { name = "${var.app_name}-listen" namespace_name = data.azurerm_eventhub_namespace.azureadevents.name eventhub_name = data.azurerm_eventhub.auditevents.name resource_group_name = data.azurerm_resource_group.events.name listen = true send = false manage = false } resource "azurerm_eventhub_authorization_rule" "onedrive_storage_access" { name = "${var.app_name}-onedrive-listen" namespace_name = data.azurerm_eventhub_namespace.azureadevents.name eventhub_name = data.azurerm_eventhub.auditevents.name resource_group_name = data.azurerm_resource_group.events.name listen = true send = false manage = false } // Additional Data Dependencies ======================================================== data "azuread_application_published_app_ids" "well_known" { } resource "azuread_service_principal" "msgraph" { // To get IDs of MicrosoftGraph APIs client_id = data.azuread_application_published_app_ids.well_known.result.MicrosoftGraph use_existing = true } data "azurerm_resource_group" "events" { name = var.eventhub_resource_group } data "azurerm_eventhub_namespace" "azureadevents" { name = var.eventhub_namespace resource_group_name = data.azurerm_resource_group.events.name } data "azurerm_eventhub" "auditevents" { name = var.eventhub_name resource_group_name = data.azurerm_resource_group.events.name namespace_name = data.azurerm_eventhub_namespace.azureadevents.name } // Additional Access for Deployment Service Principal ================================== resource "azurerm_role_assignment" "read_eventhub" { scope = data.azurerm_resource_group.events.id role_definition_name = "Reader" principal_id = module.azure_projectinit.service_principal_id skip_service_principal_aad_check = true } module "env_resources" { source = "./modules/rg-resources" for_each = { for k, v in module.azure_projectinit.resource_groups : k => v } app_name = var.app_name environment_type = var.environment_type resource_group = each.value tenant_id = data.azurerm_client_config.current.tenant_id deployer_object_id = data.azurerm_client_config.current.object_id service_principal_id = module.azure_projectinit.service_principal_id key_vault_admins = var.key_vault_admins } ----------------------------------------------------------------------------------------- https://github.com/wc-platform-engineering/od-preprovisioning-infra/blob/main/.ci/azure/output.tf >>>>>>>>>>>>> output "resource_groups" { description = "Resource groups associated with this project/environment tier" value = module.azure_projectinit.resource_groups } output "service_principal_tenant_id" { description = "Tenant ID associated with the service principal" value = module.azure_projectinit.service_principal_tenant_id } output "service_principal_id" { description = "Object ID associated with the service principal" value = module.azure_projectinit.service_principal_id } output "service_principal_secret" { description = "Secret used to authenticate with this service principal" value = module.azure_projectinit.service_principal_secret sensitive = true } output "event_hub_name" { description = "Name of the target event hub" value = data.azurerm_eventhub.auditevents.name } output "event_hub_connection_string" { description = "Primary connection string for this app to connect to event hub" value = azurerm_eventhub_authorization_rule.app_access.primary_connection_string sensitive = true } output "event_hub_connection_onedrive_string" { description = "Primary connection string for this app to connect to event hub" value = azurerm_eventhub_authorization_rule.onedrive_storage_access.primary_connection_string sensitive = true } output "key_vault_reference_id" { description = "ID to reference the key vault secret" value = module.env_resources["eastus"].sharepoint_secret_id } ------------------------------------------------------------------------------ https://github.com/wc-platform-engineering/od-preprovisioning-infra/blob/main/.ci/azure/variables.tf >>>>>>>>>>>>>>> variable "app_name" { type = string description = <<EOT Name of the application this application/deployer will be associated with. This should match unique name used to identify the application/service. EOT } variable "environment_type" { type = string description = "Environment type/name for the deployment, such as prod, dev" } variable "regions" { type = list(string) description = "Azure regions (locations) to deploy resource groups to" } variable "tags" { type = map(string) } variable "graph_api_permissions" { type = list(string) default = null } variable "eventhub_resource_group" { type = string } variable "eventhub_namespace" { type = string } variable "eventhub_name" { type = string } variable "key_vault_admins" { type = list(string) } ----------------------------------------------------------------------------------- https://github.com/wc-platform-engineering/od-preprovisioning-infra/blob/main/.ci/README.md >>>>>>> Initializing Project for Azure Deployments Use the helper script in this directory to initialize a project for any environment that it needs to be deployed into. Usage # Ensure the working directory is the root of the .ci directory cd .ci # Execute the helper script ENVIRONMENT_NAME={{environment_name}} AZURE_SUBSCRIPTION_ID={{target_azure_subscription_id}} GITLAB_ACCESS_TOKEN={{gitlab_personal_access_token}} GITLAB_PROJECT_ID={{gitlab_project_id}} bash ./init_project.sh \ -a init \ -e "$ENVIRONMENT_NAME" \ -s "$AZURE_SUBSCRIPTION_ID" \ -t "$GITLAB_ACCESS_TOKEN" \ -p "$GITLAB_PROJECT_ID" ---------------------------------------------------------------------------------------- https://github.com/wc-platform-engineering/od-preprovisioning-infra/blob/main/.ci/init_project.sh >>>>>>>>>>>>> #/usr/bin/env bash set -eo pipefail [[ "$TRACE" ]] && set -x auth_azure() { az login } list_gitlab_environments () { local response=$(curl -Ss -H "${HEADER}" "${API_URL}/environments") local environments=$(echo "${response}" | jq -r '.[].name') echo "${environments}" } create_gitlab_environment () { local name="${1}" tier="${2}" external_url="${3}" local body="name=${name}&tier=${tier}&external_url=${external_url}" local response=$(curl -Ss -X POST -H "${HEADER}" --data "${body}" "${API_URL}/environments/") echo "${response}" } list_gitlab_variables () { local result=$(curl -Ss -H "${HEADER}" "${API_URL}/variables") local variables=$(echo "${result}" | jq -r --arg scope "${SCOPE}" '.[] | select(.environment_scope==$scope) | .key' ) echo "${variables}" } update_gitlab_project() { local scope="${1}" local environments=$(list_gitlab_environments) local sp_tenant_id=$(terraform output -raw "service_principal_tenant_id") local sp_id=$(terraform output -raw "service_principal_id") local sp_secret=$(terraform output -raw "service_principal_secret") local eventhub_name=$(terraform output -raw "event_hub_name") local eventhub_conn=$(terraform output -raw "event_hub_connection_string") local eventhub_conn_onedrive_storage=$(terraform output -raw "event_hub_connection_onedrive_string") local sharepoint_secret_id=$(terraform output -raw "key_vault_reference_id") local subscription_id="$ARM_SUBSCRIPTION_ID" if [[ "${environments}" != *"${scope}"* ]]; then create_gitlab_environment "${scope}" "${TIER}" fi VARIABLES=$(list_gitlab_variables "${scope}") for var in ${AZURE_VARS[*]}; do case $var in "ARM_TENANT_ID" ) value=$sp_tenant_id protected="false" masked="false" ;; "ARM_SUBSCRIPTION_ID" ) value=$subscription_id protected="false" masked="false" ;; "ARM_CLIENT_ID" ) value=$sp_id protected="false" masked="false" ;; "ARM_CLIENT_SECRET" ) value=$sp_secret protected="true" masked="true" ;; "*" ) value="" echo "Unknown variable name: $var" ;; esac set_gitlab_variable "${var}" "${value}" "${protected}" "${masked}" done set_gitlab_variable "TF_VAR_audit_event_hub_connection" "${eventhub_conn}" "true" "false" set_gitlab_variable "TF_VAR_audit_event_hub_onedrive_connection" "${eventhub_conn_onedrive_storage}" "true" "false" set_gitlab_variable "TF_VAR_audit_event_hub_name" "${eventhub_name}" "false" "false" set_gitlab_variable "TF_VAR_app_sharepoint_secret" "${sharepoint_secret_id}" "true" "false" } set_gitlab_variable() { local key="${1}" value="${2}" protected="${3:-false}" masked="${4:-false}" if [[ "${VARIABLES}" != *"$key"* ]]; then local body="key=${key}&value=${value}&environment_scope=${SCOPE}&protected=${protected}&masked=${masked}" local response=$(curl -Ss -X POST -H "${HEADER}" --data "${body}" "${API_URL}/variables") else local body="key=${key}&value=${value}&environment_scope=${SCOPE}&protected=${protected}&masked=${masked}&filter[environment_scope]=${SCOPE}" local response=$(curl -Ss -X PUT -H "${HEADER}" --data "${body}" "${API_URL}/variables/${key}") fi echo "${response}" } terraform_init() { export TF_ADDRESS="${API_URL}/terraform/state/${SCOPE}-deployer" export TF_USERNAME="$(git config -l | grep 'user.name' | awk -F '=' '{print $NF}')" export TF_PASSWORD="${GITLAB_ACCESS_TOKEN}" terraform init \ -backend-config=address="${TF_ADDRESS}" \ -backend-config=lock_address="${TF_ADDRESS}/lock" \ -backend-config=unlock_address="${TF_ADDRESS}/lock" \ -backend-config=username="$(git config -l | grep 'user.name' | awk -F '=' '{print $NF}')" \ -backend-config=password="${TF_PASSWORD}" \ -backend-config=lock_method=POST \ -backend-config=unlock_method=DELETE \ -backend-config=retry_wait_min=5 \ -reconfigure } terraform_deploy() { local scope="${1}" pushd azure local workdir=$(mktemp -d) local tfvars_path="./vars/${scope}.tfvars" local plan_path="${workdir}/${scope}.plan" terraform_init terraform plan -var-file="${tfvars_path}" -out="${plan_path}" terraform apply "${plan_path}" } terraform_destroy() { local scope="${1}" pushd azure local tfvars_path="./vars/${scope}.tfvars" terraform_init terraform destroy -var-file="${tfvars_path}" } init_environment() { auth_azure terraform_deploy "${SCOPE}" update_gitlab_project "${SCOPE}" echo "Initialization for ${SCOPE} complete." } display_help() { echo "Usage: $0 [-a <init|clean>] [-e <prod|dev|stage|test>] [-s <string>] [-t <string>] [-p <string>]" echo echo "options:" echo "a Action to perform [init|clean]" echo "e Environment for deployment to target [prod|dev|stage|test|other]" echo "s Azure Subcription ID" echo "t GitLab Personal Access Token" echo "p GitLab Project ID" echo "h Print this Help." echo } # Handle flags while getopts ":a:p:e:s:t:" o; do case "${o}" in a) action="${OPTARG}";; p) project="${OPTARG}";; e) environment="${OPTARG}";; s) subscription="${OPTARG}";; t) token="${OPTARG}";; *) display_help; exit;; esac done if [ ! -z ${action+x} ]; then export ARM_SUBSCRIPTION_ID="${subscription}" export GITLAB_ACCESS_TOKEN="${token}" export PROJECT_ID="${project}" export API_URL="https://gitlab.whitecase.com/api/v4/projects/${PROJECT_ID}" case $environment in "prod" | "production" ) export SCOPE="prod"; export TIER="production";; "stage" | "staging" ) export SCOPE="stage"; export TIER="staging";; "test" | "testing" ) export SCOPE="test"; export TIER="testing";; "dev" | "development" ) export SCOPE="dev"; export TIER="development";; * ) export SCOPE="$environment"; export TIER="other";; esac export AZURE_VARS=("ARM_TENANT_ID" "ARM_SUBSCRIPTION_ID" "ARM_CLIENT_ID" "ARM_CLIENT_SECRET") export HEADER="PRIVATE-TOKEN: ${GITLAB_ACCESS_TOKEN}" case $action in "init" ) init_environment ;; "clean" ) terraform_destroy "${SCOPE}" ;; * ) "Invalid action specified: ${action}" ;; esac fi -------------------------------------------------------------------------------------- all these for the ci , this what we did to enhance the repo . then lets start with terraform folder : https://github.com/wc-platform-engineering/od-preprovisioning-infra/blob/main/terraform/.terraform.lock.hcl >>>>>>>>>>>>>>>>>>>>>> # This file is maintained automatically by "terraform init". # Manual edits may be lost in future updates. provider "registry.terraform.io/hashicorp/archive" { version = "2.7.1" hashes = [ "h1:62VrkalDPMKB9zerCBS4iKTbvxejwnAWn/XXYZZQWD4=", "zh:19881bb356a4a656a865f48aee70c0b8a03c35951b7799b6113883f67f196e8e", "zh:2fcfbf6318dd514863268b09bbe19bfc958339c636bcbcc3664b45f2b8bf5cc6", "zh:3323ab9a504ce0a115c28e64d0739369fe85151291a2ce480d51ccbb0c381ac5", "zh:362674746fb3da3ab9bd4e70c75a3cdd9801a6cf258991102e2c46669cf68e19", "zh:7140a46d748fdd12212161445c46bbbf30a3f4586c6ac97dd497f0c2565fe949", "zh:78d5eefdd9e494defcb3c68d282b8f96630502cac21d1ea161f53cfe9bb483b3", "zh:875e6ce78b10f73b1efc849bfcc7af3a28c83a52f878f503bb22776f71d79521", "zh:b872c6ed24e38428d817ebfb214da69ea7eefc2c38e5a774db2ccd58e54d3a22", "zh:cd6a44f731c1633ae5d37662af86e7b01ae4c96eb8b04144255824c3f350392d", "zh:e0600f5e8da12710b0c52d6df0ba147a5486427c1a2cc78f31eea37a47ee1b07", "zh:f21b2e2563bbb1e44e73557bcd6cdbc1ceb369d471049c40eb56cb84b6317a60", "zh:f752829eba1cc04a479cf7ae7271526b402e206d5bcf1fcce9f535de5ff9e4e6", ] } provider "registry.terraform.io/hashicorp/azurerm" { version = "4.57.0" constraints = "~> 4.57.0" hashes = [ "h1:NhgHn/RyZRDXMa7pEQlGv/9B+wjk48E+lvgq4asFKHs=", "zh:05e1cc7fee7829919b772ca6ce893d9c2abb3535ebff172df38f7358cdaf8f9e", "zh:30122203abc381660582f989c9e53874bd9ff93e25476a5536ea0ae37dd51f4b", "zh:4a90f008f7707d95f8f9aca90f140a9ca0e9506b0a6d436fe516de4026cacd86", "zh:6d9e114b8aed06454b71fe91a0591cc6a16cd7acde3cb36a96e4aeaec06a315a", "zh:7145c50facd9d40615fc63561ec21962feae3fa262239f9f1f1339581226104b", "zh:78d5eefdd9e494defcb3c68d282b8f96630502cac21d1ea161f53cfe9bb483b3", "zh:95f60557f1bc4210ecc3c11e2f86fe983ed7e4af19036616b605887c1195f2ac", "zh:9722b3ab879a3457588af5f0dcd65e997263affe4b829e60ecd59dbef6239e70", "zh:b891f295b018d058e8c6f841d923d5b30ba13b8208f2c20aacc70ba48c5cc0da", "zh:cb7ff113ca0bd91ab76f9f7a492d6ce9c911d6c4deb8c8e263e38322b5ff861e", "zh:ec2950bf003d29bee3fa87ab073d57fe14a4da52e9fc646fec27798b700cc8af", "zh:f69899d9e1d570a560cfa97aebec3edc2b106f2ebc15dfdee473294dd8756deb", ] } provider "registry.terraform.io/hashicorp/http" { version = "3.5.0" hashes = [ "h1:8bUoPwS4hahOvzCBj6b04ObLVFXCEmEN8T/5eOHmWOM=", "zh:047c5b4920751b13425efe0d011b3a23a3be97d02d9c0e3c60985521c9c456b7", "zh:157866f700470207561f6d032d344916b82268ecd0cf8174fb11c0674c8d0736", "zh:1973eb9383b0d83dd4fd5e662f0f16de837d072b64a6b7cd703410d730499476", "zh:212f833a4e6d020840672f6f88273d62a564f44acb0c857b5961cdb3bbc14c90", "zh:2c8034bc039fffaa1d4965ca02a8c6d57301e5fa9fff4773e684b46e3f78e76a", "zh:5df353fc5b2dd31577def9cc1a4ebf0c9a9c2699d223c6b02087a3089c74a1c6", "zh:672083810d4185076c81b16ad13d1224b9e6ea7f4850951d2ab8d30fa6e41f08", "zh:78d5eefdd9e494defcb3c68d282b8f96630502cac21d1ea161f53cfe9bb483b3", "zh:7b4200f18abdbe39904b03537e1a78f21ebafe60f1c861a44387d314fda69da6", "zh:843feacacd86baed820f81a6c9f7bd32cf302db3d7a0f39e87976ebc7a7cc2ee", "zh:a9ea5096ab91aab260b22e4251c05f08dad2ed77e43e5e4fadcdfd87f2c78926", "zh:d02b288922811739059e90184c7f76d45d07d3a77cc48d0b15fd3db14e928623", ] } provider "registry.terraform.io/hashicorp/null" { version = "3.2.4" hashes = [ "h1:hkf5w5B6q8e2A42ND2CjAvgvSN3puAosDmOJb3zCVQM=", "zh:59f6b52ab4ff35739647f9509ee6d93d7c032985d9f8c6237d1f8a59471bbbe2", "zh:78d5eefdd9e494defcb3c68d282b8f96630502cac21d1ea161f53cfe9bb483b3", "zh:795c897119ff082133150121d39ff26cb5f89a730a2c8c26f3a9c1abf81a9c43", "zh:7b9c7b16f118fbc2b05a983817b8ce2f86df125857966ad356353baf4bff5c0a", "zh:85e33ab43e0e1726e5f97a874b8e24820b6565ff8076523cc2922ba671492991", "zh:9d32ac3619cfc93eb3c4f423492a8e0f79db05fec58e449dee9b2d5873d5f69f", "zh:9e15c3c9dd8e0d1e3731841d44c34571b6c97f5b95e8296a45318b94e5287a6e", "zh:b4c2ab35d1b7696c30b64bf2c0f3a62329107bd1a9121ce70683dec58af19615", "zh:c43723e8cc65bcdf5e0c92581dcbbdcbdcf18b8d2037406a5f2033b1e22de442", "zh:ceb5495d9c31bfb299d246ab333f08c7fb0d67a4f82681fbf47f2a21c3e11ab5", "zh:e171026b3659305c558d9804062762d168f50ba02b88b231d20ec99578a6233f", "zh:ed0fe2acdb61330b01841fa790be00ec6beaac91d41f311fb8254f74eb6a711f", ] } provider "registry.terraform.io/hashicorp/random" { version = "3.7.2" hashes = [ "h1:356j/3XnXEKr9nyicLUufzoF4Yr6hRy481KIxRVpK0c=", "zh:14829603a32e4bc4d05062f059e545a91e27ff033756b48afbae6b3c835f508f", "zh:1527fb07d9fea400d70e9e6eb4a2b918d5060d604749b6f1c361518e7da546dc", "zh:1e86bcd7ebec85ba336b423ba1db046aeaa3c0e5f921039b3f1a6fc2f978feab", "zh:24536dec8bde66753f4b4030b8f3ef43c196d69cccbea1c382d01b222478c7a3", "zh:29f1786486759fad9b0ce4fdfbbfece9343ad47cd50119045075e05afe49d212", "zh:4d701e978c2dd8604ba1ce962b047607701e65c078cb22e97171513e9e57491f", "zh:78d5eefdd9e494defcb3c68d282b8f96630502cac21d1ea161f53cfe9bb483b3", "zh:7b8434212eef0f8c83f5a90c6d76feaf850f6502b61b53c329e85b3b281cba34", "zh:ac8a23c212258b7976e1621275e3af7099e7e4a3d4478cf8d5d2a27f3bc3e967", "zh:b516ca74431f3df4c6cf90ddcdb4042c626e026317a33c53f0b445a3d93b720d", "zh:dc76e4326aec2490c1600d6871a95e78f9050f9ce427c71707ea412a2f2f1a62", "zh:eac7b63e86c749c7d48f527671c7aee5b4e26c10be6ad7232d6860167f99dbb0", ] } ---------------------------------- https://github.com/wc-platform-engineering/od-preprovisioning-infra/blob/main/terraform/main.tf >>>>>>>>>>>>>>>>>>>>> terraform { required_providers { azurerm = { source = "hashicorp/azurerm" version = "~>4.57.0" } } backend "http" { } } provider "azurerm" { resource_provider_registrations = "none" features {} } locals { default_rg_name = "rg-${var.app_name}-${var.environment_type}-${var.region}" default_app_msi_name = "id-${var.app_name}-${var.environment_type}-${var.region}" key_vault_secrets = { for key, value in var.key_vault_secrets : key => "@Microsoft.KeyVault(VaultName=${var.key_vault_name};SecretName=${value})" } default_app_settings = { AUDIT_EVENT_HUB_CONNECTION = var.audit_event_hub_connection AUDIT_EVENT_HUB_ODB_CONNECTION = var.audit_event_hub_onedrive_connection AUDIT_EVENT_HUB_NAME = var.audit_event_hub_name AUDIT_EVENT_ODB_CONSUMER_GROUP = "${var.app_name}" AUDIT_EVENT_OdBMove_CONSUMER_GROUP = "odb-move" AUDIT_EVENT_ODB_STORAGE_CONSUMER_GROUP = "odb-storage" APP_GROUP_OBJECT_ID = join(",", var.app_group_object_id) APP_ONEDRIVE_GROUP_OBJECT_ID = join(",", var.app_onedrive_group_object_id) SCM_DO_BUILD_DURING_DEPLOYMENT = false FUNCTIONS_WORKER_RUNTIME = "powershell" WEBSITE_RUN_FROM_PACKAGE = "1" } app_settings = merge( local.default_app_settings, local.key_vault_secrets, var.function_app_settings ) } data "azurerm_resource_group" "default" { name = local.default_rg_name } # data "archive_file" "main_function" { >>> Now handled by you code pipeline # type = "zip" # source_dir = "${path.module}/../src/" # output_path = "${path.module}/main.zip" #} resource "azurerm_storage_account" "default" { name = module.storage_account_name.result resource_group_name = data.azurerm_resource_group.default.name location = data.azurerm_resource_group.default.location account_kind = "StorageV2" account_tier = "Standard" account_replication_type = "LRS" tags = data.azurerm_resource_group.default.tags } resource "azurerm_storage_container" "default" { name = "content" storage_account_name = azurerm_storage_account.default.name container_access_type = "private" } resource "azurerm_service_plan" "default" { name = module.service_plan_name.result resource_group_name = data.azurerm_resource_group.default.name location = data.azurerm_resource_group.default.location os_type = "Windows" sku_name = var.app_service_plan_sku tags = data.azurerm_resource_group.default.tags } data "azurerm_user_assigned_identity" "app_msi" { name = local.default_app_msi_name resource_group_name = data.azurerm_resource_group.default.name } resource "azurerm_application_insights" "default" { name = module.app_insights_name.result resource_group_name = data.azurerm_resource_group.default.name location = data.azurerm_resource_group.default.location application_type = "web" timeouts { read = "10m" } } resource "random_string" "host_id_prefix" { length = 16 special = false upper = false lower = true numeric = true } resource "azurerm_windows_function_app" "default" { name = module.function_app_name.result resource_group_name = data.azurerm_resource_group.default.name location = data.azurerm_resource_group.default.location service_plan_id = azurerm_service_plan.default.id enabled = true builtin_logging_enabled = true https_only = true functions_extension_version = "~4" key_vault_reference_identity_id = data.azurerm_user_assigned_identity.app_msi.id storage_account_name = azurerm_storage_account.default.name storage_account_access_key = azurerm_storage_account.default.primary_access_key tags = data.azurerm_resource_group.default.tags site_config { always_on = true http2_enabled = true managed_pipeline_mode = "Integrated" minimum_tls_version = 1.2 remote_debugging_enabled = false use_32_bit_worker = false websockets_enabled = false worker_count = var.function_app_worker_count application_insights_connection_string = azurerm_application_insights.default.connection_string application_insights_key = azurerm_application_insights.default.instrumentation_key application_stack { powershell_core_version = "7.4" } app_service_logs { disk_quota_mb = 50 retention_period_days = 0 } dynamic "cors" { for_each = length(var.cors) > 0 ? [1] : [] content { allowed_origins = var.cors support_credentials = true } } } app_settings = merge( local.app_settings, { "AzureFunctionsWebHost__hostid" : "${random_string.host_id_prefix.result}-production" } ) identity { type = "UserAssigned" identity_ids = [data.azurerm_user_assigned_identity.app_msi.id] } sticky_settings { app_setting_names = [ "AUDIT_EVENT_HUB_CONNECTION", "AUDIT_EVENT_HUB_ONEDRIVE_CONNECTION", "AUDIT_EVENT_HUB_NAME", "AUDIT_EVENT_ODB_CONSUMER_GROUP", "AzureFunctionsWebHost__hostid" ] } lifecycle { ignore_changes = [ tags["hidden-link: /app-insights-conn-string"], tags["hidden-link: /app-insights-instrumentation-key"], tags["hidden-link: /app-insights-resource-id"], app_settings["FUNCTIONS_EXTENSION_VERSION"], ] } } resource "azurerm_windows_function_app_slot" "staging" { name = "Staging" function_app_id = azurerm_windows_function_app.default.id functions_extension_version = "~4" builtin_logging_enabled = true https_only = true key_vault_reference_identity_id = data.azurerm_user_assigned_identity.app_msi.id storage_account_name = azurerm_storage_account.default.name storage_account_access_key = azurerm_storage_account.default.primary_access_key tags = data.azurerm_resource_group.default.tags site_config { always_on = true auto_swap_slot_name = "production" http2_enabled = true managed_pipeline_mode = "Integrated" minimum_tls_version = 1.2 remote_debugging_enabled = false use_32_bit_worker = false websockets_enabled = false worker_count = var.function_app_worker_count application_insights_connection_string = azurerm_application_insights.default.connection_string application_insights_key = azurerm_application_insights.default.instrumentation_key application_stack { powershell_core_version = "7.4" } app_service_logs { disk_quota_mb = 50 retention_period_days = 0 } } app_settings = merge( local.app_settings, { "AzureFunctionsWebHost__hostid" : "${random_string.host_id_prefix.result}-staging" } ) identity { type = "UserAssigned" identity_ids = [data.azurerm_user_assigned_identity.app_msi.id] } lifecycle { ignore_changes = [ tags["hidden-link: /app-insights-conn-string"], tags["hidden-link: /app-insights-instrumentation-key"], tags["hidden-link: /app-insights-resource-id"], app_settings["FUNCTIONS_EXTENSION_VERSION"], ] } } ------------------------------------------------------------------------------------- https://github.com/wc-platform-engineering/od-preprovisioning-infra/blob/main/terraform/names.tf >>>>>>>>>>>>>>>>>>>>>>>>>>>>>>> module "storage_account_name" { source = "github.com/wc-terraform-modules/tf-azure-resourcename?ref=v2.1.0" name = var.app_name resource_type = "azurerm_storage_account" suffixes = [var.environment_type, var.region] } module "service_plan_name" { source = "github.com/wc-terraform-modules/tf-azure-resourcename?ref=v2.1.0" name = var.app_name resource_type = "azurerm_app_service_plan" suffixes = [var.environment_type, var.region] } module "function_app_name" { source = "github.com/wc-terraform-modules/tf-azure-resourcename?ref=v2.1.0" name = var.app_name resource_type = "azurerm_function_app" suffixes = [var.environment_type, var.region] } module "app_insights_name" { source = "github.com/wc-terraform-modules/tf-azure-resourcename?ref=v2.1.0" name = var.app_name resource_type = "azurerm_application_insights" suffixes = [var.environment_type, var.region] } ----------------------------------------------------------------------------------------- https://github.com/wc-platform-engineering/od-preprovisioning-infra/blob/main/terraform/outputs.tf >>>>>>>>>> output "resource_group" { description = "Function apps resource group" value = data.azurerm_resource_group.default.name } output "function_app_name" { description = "Function app name" value = azurerm_windows_function_app.default.name } output "archive_file" { description = "Archive file for the zip deployment process" value = data.archive_file.main_function.output_path } ------------------------------------------------------------------------------------------------ https://github.com/wc-platform-engineering/od-preprovisioning-infra/blob/main/terraform/variables.tf >>>>>>>>>>>>>>>>>>>>>>>>>>>>> /* Terraform input variables */ variable "app_name" { type = string description = "The parent/root name for this deployment. This will be applied as a prefix to all resources created." } variable "environment_type" { type = string description = "Type/Tier of environment this is deploying" } variable "region" { type = string description = "Deployment location in Azure for resources" } variable "app_service_plan_sku" { type = string description = "SKU for App Service Plan" default = "S1" } variable "function_app_worker_count" { type = string description = "Number of workers for the function app" default = "1" } variable "audit_event_hub_connection" { type = string description = "Connection string for Event Hub" } variable "audit_event_hub_onedrive_connection" { description = "The connection string for the OneDrive Storage audit event hub" type = string } variable "audit_event_hub_name" { type = string description = "Name of Event Hub to trigger on" } variable "app_tenant_name" { type = string } variable "app_group_object_id" { type = list(string) default = [ ] nullable = false } variable "app_onedrive_group_object_id" { type = list(string) default = [ ] nullable = false } variable "key_vault_name" { type = string description = "Name of the Key Vault that stores app secrets" } variable "key_vault_secrets" { type = map(string) description = "Map of secrets that are pulled from the Key Vault for use in the function app settings." default = { APP_TENANT_NAME = "TenantName" APP_SHAREPOINT_USER = "SharepointUser" APP_SHAREPOINT_SECRET = "SharepointSecret" } } variable "function_app_settings" { type = map(string) description = "Map of application settings to apply to the function app. The keys in this map will be available to the function app as environment variables." default = {} } variable "app_sharepoint_user" { type = string } variable "app_sharepoint_secret" { type = string sensitive = true } variable "cors" { type = list(string) default = [] } ----------------------------------------------------------------------------------------- this is the whole terraform files ,,,,, then lets start with vars folder : this is deffrent than the ones from ci folder : https://github.com/wc-platform-engineering/od-preprovisioning-infra/blob/main/vars/dev.tfvars >>>>>>>>>>>>>>>>> /* Development Environment Input Variables */ app_name = "onedrivepreprov" region = "eastus" environment_type = "dev" app_tenant_name = "boosie" app_group_object_id = ["e6132609-1100-4e99-91f8-f90f8299f9d0","b29b928e-d0f5-43e0-bcda-131f89e12251"] app_sharepoint_user = "3l2l-teams-app-svc@3law2law.com" # aad-acl-onedrive-storage-80gb aad-acl-onedrive-storage-40gb app_onedrive_group_object_id=["a229ac97-764d-4b05-b054-a42775b79f4c","3750e7db-2e8b-40eb-9506-b17cfd863b5e"] key_vault_name = "kv-onedr-dev-eastu" key_vault_secrets = { APP_TENANT_NAME = "TenantName" APP_SHAREPOINT_USER = "SharepointUser" APP_SHAREPOINT_SECRET = "SharepointSecret" } cors = ["https://portal.azure.com"] ---------------------------------------------------------------------------------------- https://github.com/wc-platform-engineering/od-preprovisioning-infra/blob/main/vars/prod.tfvars >>>>>> /* Production Environment Input Variables */ app_name = "onedrivepreprov" region = "eastus" environment_type = "prod" app_tenant_name = "whitecasempsa" app_group_object_id = ["4ab9fdf2-cd4d-4a90-8b72-1f4b0cf5a517","097f2f9c-9337-49ae-8090-b7a120f34a7d"] app_sharepoint_user = "spo-func-svc@whitecase.com" app_onedrive_group_object_id=[""] key_vault_name = "kv-onedr-prod-eastu" key_vault_secrets = { APP_TENANT_NAME = "TenantName" APP_SHAREPOINT_USER = "SharepointUser" APP_SHAREPOINT_SECRET = "SharepointSecret" } ----------------------------------------------------------------------------------------- then the pipline for this project : https://github.com/wc-platform-engineering/od-preprovisioning-infra/blob/main/.github/workflows/infra-terraform.yml >>>>>>>>> name: Infrastructure Deployment on: # push: # branches: # - main # - dev workflow_dispatch: jobs: terraform-validate: runs-on: ubuntu-latest steps: - uses: actions/checkout@v3 - name: Generate GitHub App token id: app-token uses: actions/create-github-app-token@v2 with: app-id: ${{ vars.GHAPP_TERRAFORM_READER_APP_ID }} private-key: ${{ secrets.GHAPP_TERRAFORM_READER_PRIVATE_KEY }} owner: ${{ vars.GHAPP_TERRAFORM_READER_OWNER }} - name: Authenticate to git shell: bash run: | echo "${{ steps.app-token.outputs.token }}" | gh auth login --with-token gh auth setup-git - name: Setup Terraform uses: hashicorp/setup-terraform@v2 with: terraform_version: 1.10.5 - name: Terraform Init working-directory: terraform/ run: | terraform init \ -backend-config="storage_account_name=${{ secrets.AZURE_TF_STORAGE_ACCOUNT }}" \ -backend-config="container_name=${{ secrets.AZURE_TF_CONTAINER }}" \ -backend-config="key=terraform.tfstate" - name: Terraform Validate working-directory: terraform/ run: terraform validate terraform-plan: needs: terraform-validate runs-on: ubuntu-latest outputs: commit_sha: ${{ github.sha }} steps: - uses: actions/checkout@v3 - name: Generate GitHub App token id: app-token uses: actions/create-github-app-token@v2 with: app-id: ${{ vars.GHAPP_TERRAFORM_READER_APP_ID }} private-key: ${{ secrets.GHAPP_TERRAFORM_READER_PRIVATE_KEY }} owner: ${{ vars.GHAPP_TERRAFORM_READER_OWNER }} - name: Authenticate to git shell: bash run: | echo "${{ steps.app-token.outputs.token }}" | gh auth login --with-token gh auth setup-git - uses: hashicorp/setup-terraform@v2 with: terraform_version: 1.10.5 - name: Terraform Init working-directory: terraform/ run: | terraform init \ -backend-config="storage_account_name=${{ secrets.AZURE_TF_STORAGE_ACCOUNT }}" \ -backend-config="container_name=${{ secrets.AZURE_TF_CONTAINER }}" \ -backend-config="key=terraform.tfstate" - name: Debug vars folder run: | echo "Current branch: $GITHUB_REF_NAME" echo "Listing vars folder:" ls -la vars/ echo "Listing terraform folder:" ls -la terraform/ - name: Terraform Plan working-directory: terraform/ run: | if [[ "${GITHUB_REF_NAME}" == "main" ]]; then VAR_FILE="../vars/dev.tfvars" else VAR_FILE="../vars/${GITHUB_REF_NAME}.tfvars" fi echo "Using variable file: $VAR_FILE" terraform plan -var-file="$VAR_FILE" -out=tfplan - name: Upload Plan Artifact uses: actions/upload-artifact@v4 with: name: tfplan path: terraform/tfplan # terraform-apply: # needs: terraform-plan # if: github.sha == needs.terraform-plan.outputs.commit_sha # runs-on: ubuntu-latest # steps: # - uses: actions/checkout@v3 # - name: Generate GitHub App token # id: app-token # uses: actions/create-github-app-token@v2 # with: # app-id: ${{ vars.GHAPP_TERRAFORM_READER_APP_ID }} # private-key: ${{ secrets.GHAPP_TERRAFORM_READER_PRIVATE_KEY }} # owner: ${{ vars.GHAPP_TERRAFORM_READER_OWNER }} # - name: Authenticate to git # shell: bash # run: | # echo "${{ steps.app-token.outputs.token }}" | gh auth login --with-token # gh auth setup-git # - uses: hashicorp/setup-terraform@v2 # with: # terraform_version: 1.10.5 # - name: Download Plan Artifact # uses: actions/download-artifact@v4 # with: # name: tfplan # path: terraform/ # - name: Azure Login via OIDC # uses: azure/login@v1 # with: # client-id: ${{ secrets.AZURE_CLIENT_ID }} # tenant-id: ${{ secrets.AZURE_TENANT_ID }} # subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }} # - name: Terraform Init # working-directory: terraform/ # run: | # terraform init \ # -backend-config="storage_account_name=${{ secrets.AZURE_TF_STORAGE_ACCOUNT }}" \ # -backend-config="container_name=${{ secrets.AZURE_TF_CONTAINER }}" \ # -backend-config="key=terraform.tfstate" # - name: Terraform Apply # working-directory: terraform/ # run: terraform apply -input=false tfplan -------- as you see i commented out the apply step just for sopping any unexpected deployments ..........................................................................
ok ,,, add those paths ,,,, i will do the dev one and the staging and prod with be the same mostly ,,, lets start with the dev: >>>>> the path first : https://github.com/wc-platform-engineering/terragrunt-onedrive-infra/blob/main/infrastructure/deployments/dev/eastus/sample_unit/primary/.terraform.lock.hcl then the code  >>>> # This file is maintained automatically by "tofu init".
# Manual edits may be lost in future updates.

provider "registry.opentofu.org/hashicorp/http" {
  version     = "3.5.0"
  constraints = ">= 3.5.0"
  hashes = [
    "h1:eClUBisXme48lqiUl3U2+H2a2mzDawS9biqfkd9synw=",
    "zh:0a2b33494eec6a91a183629cf217e073be063624c5d3f70870456ddb478308e9",
    "zh:180f40124fa01b98b3d2f79128646b151818e09d6a1a9ca08e0b032a0b1e9cb1",
    "zh:3e29e1de149dc10bf78620526c7cb8c62cd76087f5630dfaba0e93cda1f3aa7b",
    "zh:4420950200cf86042ec940d0e2c9b7c89966bf556bf8038ba36217eae663bca5",
    "zh:5d1f7d02109b2e2dca7ec626e5563ee765583792d0fd64081286f16f9433bd0d",
    "zh:8500b138d338b1994c4206aa577b5c44e1d7260825babcf43245a7075bfa52a5",
    "zh:b42165a6c4cfb22825938272d12b676e4a6946ac4e750f85df870c947685df2d",
    "zh:b919bf3ee8e3b01051a0da3433b443a925e272893d3724ee8fc0f666ec7012c9",
    "zh:d13b81ea6755cae785b3e11634936cdff2dc1ec009dc9610d8e3c7eb32f42e69",
    "zh:f1c9d2eb1a6b618ae77ad86649679241bd8d6aacec06d0a68d86f748687f4eb3",
  ]
}

provider "registry.opentofu.org/hashicorp/local" {
  version     = "2.6.1"
  constraints = ">= 2.5.3"
  hashes = [
    "h1:+XfQ7VmNtYMp0eOnoQH6cZpSMk12IP1X6tEkMoMGQ/A=",
    "zh:0416d7bf0b459a995cf48f202af7b7ffa252def7d23386fc05b34f67347a22ba",
    "zh:24743d559026b59610eb3d9fa9ec7fbeb06399c0ef01272e46fe5c313eb5c6ff",
    "zh:2561cdfbc90090fee7f844a5cb5cbed8472ce264f5d505acb18326650a5b563f",
    "zh:3ebc3f2dc7a099bd83e5c4c2b6918e5b56ec746766c58a31a3f5d189cb837db5",
    "zh:490e0ce925fc3848027e10017f960e9e19e7f9c3b620524f67ce54217d1c6390",
    "zh:bf08934295877f831f2e5f17a0b3ebb51dd608b2509077f7b22afa7722e28950",
    "zh:c298c0f72e1485588a73768cb90163863b6c3d4c71982908c219e9b87904f376",
    "zh:cedbaed4967818903ef378675211ed541c8243c4597304161363e828c7dc3d36",
    "zh:edda76726d7874128cf1e182640c332c5a5e6a66a053c0aa97e2a0e4267b3b92",
  ]
}

provider "registry.opentofu.org/hashicorp/random" {
  version     = "3.7.2"
  constraints = ">= 3.7.2"
  hashes = [
    "h1:cFGCdxTlsrteTiaOV/iOQdql7eJkD3F/vtJxenkj9IE=",
    "zh:2ffeb1058bd7b21a9e15a5301abb863053a2d42dffa3f6cf654a1667e10f4727",
    "zh:519319ed8f4312ed76519652ad6cd9f98bc75cf4ec7990a5684c072cf5dd0a5d",
    "zh:7371c2cc28c94deb9dba62fbac2685f7dde47f93019273a758dd5a2794f72919",
    "zh:9b0ac4c1d8e36a86b59ced94fa517ae9b015b1d044b3455465cc6f0eab70915d",
    "zh:c6336d7196f1318e1cbb120b3de8426ce43d4cacd2c75f45dba2dbdba666ce00",
    "zh:c71f18b0cb5d55a103ea81e346fb56db15b144459123f1be1b0209cffc1deb4e",
    "zh:d2dc49a6cac2d156e91b0506d6d756809e36bf390844a187f305094336d3e8d8",
    "zh:d5b5fc881ccc41b268f952dae303501d6ec9f9d24ee11fe2fa56eed7478e15d0",
    "zh:db9723eaca26d58c930e13fde221d93501529a5cd036b1f167ef8cff6f1a03cc",
    "zh:fe3359f733f3ab518c6f85f3a9cd89322a7143463263f30321de0973a52d4ad8",
  ]
}                                                                                                                                    ---------------------------------------------------------------------------------------                  https://github.com/wc-platform-engineering/terragrunt-onedrive-infra/blob/main/infrastructure/deployments/dev/eastus/sample_unit/primary/deployment.yaml     >>>>>>>>>>>>>>>>>>>>>>>>>>>>>>  ---
# This file contains deployment-specific configurations that will be applied to any deployment
# inside this directory. This is the most granular scope available 
# Any variable set at this scope will overwrite variables of the same name set at the
# defaults, tenant, subscription, project, environment, and location scopes.                      ------------------------------------------------------------------------------------------           https://github.com/wc-platform-engineering/terragrunt-onedrive-infra/blob/main/infrastructure/deployments/dev/eastus/sample_unit/primary/terragrunt.hcl    >>>>>      include "root" {
  path = find_in_parent_folders("root.hcl")
}

include "sample_unit" {
  path = "${get_repo_root()}/infrastructure/common/sample_unit.hcl"
}                                                                                                                                            -------------------------------------------------------------------------------------                    https://github.com/wc-platform-engineering/terragrunt-onedrive-infra/blob/main/infrastructure/deployments/dev/eastus/sample_unit/unit.yaml   >>>>>>>                 ---
# Use this file for configuring variables that should apply to all deployments of this unit.
# Any variable set at this scope will overwrite variables of the same name set at the
# defaults, tenant, subscription, project, environment, and location scopes.

# If the multiple instances of the same unit are deployed, this file can be used to set
# common configurations for all of them to share. If unique configurations are needed
# per deployment, then you would use the lower deployment.yaml file for more granular
# control.                                                                                                                              ---------------------------------------------------------------------------------------                                  https://github.com/wc-platform-engineering/terragrunt-onedrive-infra/blob/main/infrastructure/deployments/dev/eastus/location.yaml >>>>>      ---
# Use this file for configuring variables that should apply to the all deployments in this location.
# Any variable set at this scope will overwrite variables of the same name set at the
# defaults, tenant, subscription, project, and environment scopes.

# The location variable is used to set the location used for all resources within
# this scope.
# If location is not explicitly set in any configuration then it will be set automatically
# using the name of the "location" directory the deployment runs under. It is recommended
# to explicitly set this to prevent unintended changes from occurring later if
# project structure changes.
# This variable is used for tagging.
location: eastus                  
.................................................................... 
and here some content of read me file : OneDrive PreProvisioning Update OneDrive Storage Getting started To make it easy for you to get started with GitLab, here's a list of recommended next steps. Already a pro? Just edit this README.md and make it your own. Want to make it easy? Use the template at the bottom! Add your files Create or upload files Add files using the command line or push an existing Git repository with the following command: cd existing_repo git remote add origin https://gitlab.whitecase.com/application-development/onedrive-preprovisioning.git git branch -M main git push -uf origin main Integrate with your tools Set up project integrations Collaborate with your team Invite team members and collaborators Create a new merge request Automatically close issues from merge requests Enable merge request approvals Automatically merge when pipeline succeeds Test and Deploy Use the built-in continuous integration in GitLab. Get started with GitLab CI/CD Analyze your code for known vulnerabilities with Static Application Security Testing(SAST) Deploy to Kubernetes, Amazon EC2, or Amazon ECS using Auto Deploy Use pull-based deployments for improved Kubernetes management Set up protected environments Editing this README When you're ready to make this README your own, just edit this file and use the handy template below (or feel free to structure it however you want - this is just a starting point!). Thank you to makeareadme.com for this template. Suggestions for a good README Every project is different, so consider which of these sections apply to yours. The sections used in the template are suggestions for most open source projects. Also keep in mind that while a README can be too long and detailed, too long is better than too short. If you think your README is too long, consider utilizing another form of documentation rather than cutting out information. Name Choose a self-explaining name for your project. Description Let people know what your project can do specifically. Provide context and add a link to any reference visitors might be unfamiliar with. A list of Features or a Background subsection can also be added here. If there are alternatives to your project, this is a good place to list differentiating factors. Badges On some READMEs, you may see small images that convey metadata, such as whether or not all the tests are passing for the project. You can use Shields to add some to your README. Many services also have instructions for adding a badge. Visuals Depending on what you are making, it can be a good idea to include screenshots or even a video (you'll frequently see GIFs rather than actual videos). Tools like ttygif can help, but check out Asciinema for a more sophisticated method. Installation Within a particular ecosystem, there may be a common way of installing things, such as using Yarn, NuGet, or Homebrew. However, consider the possibility that whoever is reading your README is a novice and would like more guidance. Listing specific steps helps remove ambiguity and gets people to using your project as quickly as possible. If it only runs in a specific context like a particular programming language version or operating system or has dependencies that have to be installed manually, also add a Requirements subsection. Usage Use examples liberally, and show the expected output if you can. It's helpful to have inline the smallest example of usage that you can demonstrate, while providing links to more sophisticated examples if they are too long to reasonably include in the README. Support Tell people where they can go to for help. It can be any combination of an issue tracker, a chat room, an email address, etc. Roadmap If you have ideas for releases in the future, it is a good idea to list them in the README. Contributing State if you are open to contributions and what your requirements are for accepting them. For people who want to make changes to your project, it's helpful to have some documentation on how to get started. Perhaps there is a script that they should run or some environment variables that they need to set. Make these steps explicit. These instructions could also be useful to your future self. You can also document commands to lint the code or run tests. These steps help to ensure high code quality and reduce the likelihood that the changes inadvertently break something. Having instructions for running tests is especially helpful if it requires external setup, such as starting a Selenium server for testing in a browser. Authors and acknowledgment Show your appreciation to those who have contributed to the project. License For open source projects, say how it is licensed. Project status If you have run out of energy or time for your project, put a note at the top of the README saying that development has slowed down or stopped completely. Someone may choose to fork your project or volunteer to step in as a maintainer or owner, allowing your project to keep going. You can also make an explicit request for maintainers. ,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,
ok so let me share that then we can do the whole thing a gain so it will align with the  repo templet which is a standered we ganna follow in the company :                                          https://github.com/wc-platform-engineering/terragrunt-onedrive-infra  this is the repo path 
..........

lets sart with workflow : https://github.com/wc-platform-engineering/terragrunt-onedrive-infra/blob/main/.github/workflows/terragrunt-pipeline-multi-env.yml   >>>>>>>>>>>> name: Terragrunt PR Plan

permissions:
  contents: read
  pull-requests: write
  issues: write

on:
  pull_request:

env:
  ARM_CLIENT_ID: ${{ secrets.ARM_CLIENT_ID_34B12D2D_E28C_43B5_A33B_12DA8579925D }}
  ARM_SUBSCRIPTION_ID: ${{ secrets.ARM_SUBSCRIPTION_ID_34B12D2D_E28C_43B5_A33B_12DA8579925D }}
  ARM_TENANT_ID: ${{ secrets.ARM_TENANT_ID_34B12D2D_E28C_43B5_A33B_12DA8579925D }}
  ARM_RESOURCE_GROUP_NAME: ${{ secrets.ARM_RESOURCE_GROUP_NAME_34B12D2D_E28C_43B5_A33B_12DA8579925D }}
  ARM_CLIENT_CERTIFICATE_PASSWORD: ${{ secrets.ARM_CLIENT_CERTIFICATE_PASSWORD_34B12D2D_E28C_43B5_A33B_12DA8579925D }}
  ARM_CLIENT_CERTIFICATE: ${{ secrets.ARM_CLIENT_CERTIFICATE_34B12D2D_E28C_43B5_A33B_12DA8579925D }}
  GHAPP_TERRAFORM_READER_PRIVATE_KEY: ${{ secrets.GHAPP_TERRAFORM_READER_PRIVATE_KEY }}
  GHAPP_TERRAFORM_READER_APP_ID: ${{ vars.GHAPP_TERRAFORM_READER_APP_ID }}
  GHAPP_TERRAFORM_READER_OWNER: ${{ vars.GHAPP_TERRAFORM_READER_OWNER }}

jobs:
  terragrunt-environment:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        environment: [dev, staging, prod]
    outputs:
      summary_artifact_id: ${{ steps.upload_summaries.outputs.artifact-id }}

    steps:
      - uses: actions/checkout@v5
        with:
            fetch-depth: 0

      - name: Detect changes
        id: detect
        uses: wc-workflow-actions/detect-terraform-changes@v1.0.0
        with:
          environment: ${{ matrix.environment }}
          environment_path: infrastructure/deployments/${{ matrix.environment }}

      - name: Terragrunt plan
        id: plan
        if: steps.detect.outputs.has_changes == 'true'
        uses: wc-workflow-actions/create-terragrunt-plan@v1.0.2
        with:
          environment: ${{ matrix.environment }}
          modules: ${{ steps.detect.outputs.modules }}

      - name: Comment PR
        uses: wc-workflow-actions/write-github-tf-plan@v1.0.1
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          repository: ${{ github.repository }}
          pr_number: ${{ github.event.pull_request.number }}
          run_id: ${{ github.run_id }}
          environment: ${{ matrix.environment }}
          run_attempt: ${{ github.run_attempt }}
          summary_artifact_id: ${{ steps.plan.outputs.artifact-id }}                                      -------------------------------------------------------------------------------------                    https://github.com/wc-platform-engineering/terragrunt-onedrive-infra/blob/main/infrastructure/common/README.md >>>>>      Common Terragrunt Configurations
This directory contains shared .hcl configuration files used for units within this project. Most, if not all, child terragrunt configuration will rely on one or more of these shared configurations to reduce the complexity at the child level.

Defaults for most settings will be applied here. If a specific child terragrunt needs to vary from these defaults they may be defined at the child terragrunt configuration directly.

Context File
This section describes the purpose, usage, and maintenance of the Terragrunt context.hcl file, which provides a unified “context” across multiple Terragrunt units. By centralizing context, teams can define variables in various scopes—such as project, environment, or location—and merge them reliably.

Overview
The context.hcl file is intended to be a single place where variables and configurations are loaded in a flexible, hierarchical manner. This allows Terragrunt units to refer to a consistent set of variables (the “context”), no matter where they are in the project hierarchy.

Key idea
Variables may be defined at multiple scopes (e.g., Project, Environment, Location, Deployment, Unit).
Merging is done in order of broadest scope to most granular.
If duplicates exist, the most granular scope “wins.”
Why it’s helpful
Avoids scattering shared variables across many files.
Promotes DRY (don’t-repeat-yourself) patterns by letting you define a variable once in a higher-level scope, then only override it when you need something different.
Using context variables in units
In your Terragrunt unit (e.g., terragrunt.hcl in a module directory), reference the context file like so:

locals {
  # Read the context.hcl file at its consistent path
  _context_data = read_terragrunt_config("${get_repo_root()}/infrastructure/common/context.hcl").locals
  context       = {for k, v in local._context_data : k => v if !startswith(k, "_") && v != null}

  # Now you can reference any variable in `context.hcl` via local.context
  # Example: referencing a variable called azure_tenant_id
  azure_tenant_id = local.context.azure_tenant_id
}
Once loaded, any variable you define in context.hcl’s locals block is accessible using local.context.<variable_name> in your Terragrunt module.

Note: Variables prefixed with _ in context.hcl are considered internal and not intended for direct consumption (they’re used for intermediate calculations or lookups).

Default scopes and merging
context.hcl uses a sequence of steps to find and load variables from each scope (e.g., project.yaml, environment.yaml, deployment.yaml), then merges them together. The load order looks like this:

Project
Defaults
Parent
Environment
Subscription
Location
Deployment
Unit
If the same variable is defined in multiple scopes, the one from the latter scope in the list above takes precedence.

Example merge
Suppose project.yaml sets project_name = "MyProject" and environment.yaml also sets project_name = "DevProject".

Because environment.yaml is loaded after project.yaml, the final value you see in context.hcl would be "DevProject".

Adding or Removing Variables
You may need to add or remove variables in response to new project requirements. Here’s the recommended workflow:

Add the variable to the relevant scope file (e.g., environment.yaml or deployment.yaml).

Expose the variable in context.hcl by adding a line like:

application_owner = lookup(local._ctx, "application_owner", null)
Consume the variable in any Terragrunt unit by referencing local.context.application_owner.

This ensures that no matter which scope the variable originates from, it’s accessible to all modules once it’s included in the context.hcl.

Contributing
When building these shared configurations, ensure that the configuration is dynamic enough so that it can be reused across multiple environments, otherwise, create directly within the context of the child that will consume it or notate the specific purpose of the shared configuration so that future users are aware that it might not fit their expected needs.                                                                                      ------------------------------------------------------------------------------------------             https://github.com/wc-platform-engineering/terragrunt-onedrive-infra/blob/main/infrastructure/common/context.hcl   >>>>>>     # --------------------------------------------------------------------------------------
# Terragrunt Shared Deployment Context
#
# This file contains a shared (local) variable configuration to be used by all units
# defined within the project. Please see the README.md file for detailed documentation.
#
# Variables prefixed with '_' are not intended to be directly referenced in other files.
# --------------------------------------------------------------------------------------

locals {
// CORE VARIABLES - DO NOT CHANGE ----------------------------------------------------
  # Directory Variables
  repo_root_dir      = get_repo_root()                           # Root of the Git Repository
  infrastructure_dir = "${local.repo_root_dir}/infrastructure"   # Parent Terragrunt Directory
  deployment_dir     = get_original_terragrunt_dir()             # Deployment Terragrunt Directory
  deployments_dir    = "${local.infrastructure_dir}/deployments" # Root directory of deployment configurations
  modules_dir        = "${local.infrastructure_dir}/modules"     # Root directory of terraform modules
  common_dir         = "${local.infrastructure_dir}/common"      # Directory containing shared .hcl configurations
  utils_dir          = "${local.common_dir}/utils"               # Directory containing helper utilities/scripts

  # Split the relative path from ./infrastructure/deployments/ to the child terragrunt.hcl directory
  _relative_deployment_path  = trimprefix(trimprefix(local.deployment_dir, local.deployments_dir), "/")
  _sanitized_deployment_path = replace(local._relative_deployment_path, "-", "_")

  # Convert the sanitized deployment path into a unique name for the TF state
  stack_name = lower(replace(local._sanitized_deployment_path, "/", "-"))
  _stack_parts = split("-", local.stack_name)
  meta_environment   = local._stack_parts[0]
  meta_location      = local._stack_parts[1]
  meta_unit          = local._stack_parts[2]
  meta_deployment_id = local._stack_parts[3]

  # Helper Scripts/Utilities
  _tgutil_available = can(run_cmd("--terragrunt-quiet", "tgutil", "--help"))
  tgutil_bin       = local._tgutil_available ? "tgutil" : "${local.utils_dir}/tgutil"
  _search_command   = ["--terragrunt-quiet", local.tgutil_bin, "find-in-parent", "-d", local.deployment_dir, "-r", local.repo_root_dir, "-t"]

 # Load all variable scope files - loads in this order, last to load takes precedence for duplicate keys
  _scope_files = [
    "defaults.yaml",
    "tenant.yaml",
    "subscription.yaml",
    "environment.yaml",
    "project.yaml",
    "location.yaml",
    "unit.yaml",
    "deployment.yaml",
  ]
  _scope_maps = [
    for file_name in local._scope_files : try(
      yamldecode(
        file(
          try(
            run_cmd(concat(local._search_command, [file_name])...),
            "" # fallback if run_cmd fails, likely due to file not existing in the current scope
          )
        )
      ),
      {}
    )
  ]
  _ctx = merge(local._scope_maps...)

  # Terraform/Opentofu Configuration
  gitlab_api_base_url     = get_env("CI_API_V4_URL", "https://gitlab.whitecase.com/api/v4")
  gitlab_project_id       = lookup(local._ctx, "gitlab_project_id", null)
  terraform_base_url      = "${local.gitlab_api_base_url}/projects/${local.gitlab_project_id}/terraform/state"
  terraform_state_address = "${local.terraform_base_url}/${local.stack_name}"

  # Commonly Required Metadata Variables
  tenant          = lookup(local._ctx, "tenant_name", "none") # friendly name, not to be confused with azure_tenant_id
  subscription    = lookup(local._ctx, "subscription_name", "none") # friendly name, not to be confused with azure_subscription_id
  project_name    = lookup(local._ctx, "project_name", "none")
  project_owner   = lookup(local._ctx, "project_owner", null)
  project_contact = lookup(local._ctx, "project_contact", null)
  environment     = lookup(local._ctx, "environment", local.meta_environment)
  location        = lookup(local._ctx, "location", local.meta_location)
  default_tags = {
    environment          = local.environment
    location             = local.location
    project_name         = local.project_name
    project_owner        = local.project_owner
    project_contact      = local.project_contact
    deployment_id        = local.deployment_id
    managed_by_terraform = true
    tfstate              = local.stack_name
  }
  tags          = merge(local.default_tags, lookup(local._ctx, "tags", {}))
  deployments   = lookup(local._ctx, "deployments", null)
  deployment_id = lookup(local._ctx, "deployment_id", local.meta_deployment_id)

  # Common Azure Variables
  azure_tenant_id               = lookup(local._ctx, "azure_tenant_id", null)
  tenant_id                     = local.azure_tenant_id
  azure_subscription_id         = lookup(local._ctx, "azure_subscription_id", null)
  subscription_id               = local.azure_subscription_id
  azure_default_subscription_id = lookup(local._ctx, "azure_default_subscription_id", null) # Fallback for resources that are not subscription-scoped

  // END CORE VARIABLES - DO NOT CHANGE ------------------------------------------------
}                                                                                                                                           -------------------------------------------------------------------------------------                         https://github.com/wc-platform-engineering/terragrunt-onedrive-infra/blob/main/infrastructure/common/sample_unit.hcl  >>>>>>>>>>>>>>>>>>>>>>                                                                                         # This configuration is shared with different deployments of the metadata unit.

locals {
  _context_data = read_terragrunt_config("${get_repo_root()}/infrastructure/common/context.hcl").locals
  context       = {for k, v in local._context_data : k => v if !startswith(k, "_") && v != null}
}

terraform {
  source = "github.com/wc-terraform-modules/tf-azure-resourcename?ref=v2.1.0"
    
  before_hook "render_current_config" {
    commands    = get_terraform_commands_that_need_locking()
    execute     = ["terragrunt", "render"]
    working_dir = get_original_terragrunt_dir()
  }
}

# This is where a provider would be configured, but in this example there is no provider
# required. Typically you'd set your configuration for azurerm, azuread, etc.. here.
generate "providers" {
  path = "providers.tf"
  if_exists = "overwrite_terragrunt"
  contents = <<-EOF
  EOF
}

inputs = {
  name          = local.context.project_name
  suffixes      = [local.context.environment, local.context.location]
  resource_type = "azurerm_storage_account"
}                                                                                                                                            ---------------------------------------------------------------------------------------                  and this empty file : https://github.com/wc-platform-engineering/terragrunt-onedrive-infra/blob/main/infrastructure/common/utils/tgutil   >>>> i think its for future enhancements for the template .                                                                              ---------------------------------------------------------------------------------------          then : https://github.com/wc-platform-engineering/terragrunt-onedrive-infra/blob/main/infrastructure/deployments/dev/environment.yaml  >>>>>>>>>>>>>>>>>>       ---
# Use this file for configuring variables that should apply to the all deployments in this environment.
# Any variable set at this scope will overwrite variables of the same name set at the
# defaults, tenant, subscription, and project scopes.

# The environment variable is used to set the environment name for all resources within
# this scope.
# If environment is not explicitly set in any configuration then it will be set automatically
# using the name of the "environment" directory the deployment runs under. It is recommended
# to explicitly set this to prevent unintended changes from occurring later if
# project structure changes.
# This variable is used for tagging.
environment: dev

# The azure_subscription_id defines the Azure subscription that resources will be deployed
# to. This is used over the azure_default_subscription_id when it is set.
azure_subscription_id: 34b12d2d-e28c-43b5-a33b-12da8579925d # sub-plateng-dev-primary


# This variable is needed to replace the implicit "tenant" provided when using the larger
# directory structure used for mono-repo style repositories. We want to retain the same
# state name structure so we have to explicitly provide it so we don't lose the context.
subscription_name: plateng-dev                                                                                          ------------------------------------------------------------------------------------                       https://github.com/wc-platform-engineering/terragrunt-onedrive-infra/blob/main/infrastructure/deployments/prod/environment.yaml    >>>>>                  ---
# Use this file for configuring variables that should apply to the all deployments in this environment.
# Any variable set at this scope will overwrite variables of the same name set at the
# defaults, tenant, subscription, and project scopes.

# The environment variable is used to set the environment name for all resources within
# this scope.
# If environment is not explicitly set in any configuration then it will be set automatically
# using the name of the "environment" directory the deployment runs under. It is recommended
# to explicitly set this to prevent unintended changes from occurring later if
# project structure changes.
# This variable is used for tagging.
environment: prod

# The azure_subscription_id defines the Azure subscription that resources will be deployed
# to. This is used over the azure_default_subscription_id when it is set.
azure_subscription_id: 34b12d2d-e28c-43b5-a33b-12da8579925d # sub-plateng-dev-primary


# This variable is needed to replace the implicit "tenant" provided when using the larger
# directory structure used for mono-repo style repositories. We want to retain the same
# state name structure so we have to explicitly provide it so we don't lose the context.
subscription_name: plateng-dev                                                                                           ------------------------------------------------------------------------------------------        https://github.com/wc-platform-engineering/terragrunt-onedrive-infra/blob/main/infrastructure/deployments/staging/environment.yaml    >>>>>>>>>>>>>>>          ---
# Use this file for configuring variables that should apply to the all deployments in this environment.
# Any variable set at this scope will overwrite variables of the same name set at the
# defaults, tenant, subscription, and project scopes.

# The environment variable is used to set the environment name for all resources within
# this scope.
# If environment is not explicitly set in any configuration then it will be set automatically
# using the name of the "environment" directory the deployment runs under. It is recommended
# to explicitly set this to prevent unintended changes from occurring later if
# project structure changes.
# This variable is used for tagging.
environment: staging

# The azure_subscription_id defines the Azure subscription that resources will be deployed
# to. This is used over the azure_default_subscription_id when it is set.
azure_subscription_id: 34b12d2d-e28c-43b5-a33b-12da8579925d # sub-plateng-dev-primary


# This variable is needed to replace the implicit "tenant" provided when using the larger
# directory structure used for mono-repo style repositories. We want to retain the same
# state name structure so we have to explicitly provide it so we don't lose the context.
subscription_name: plateng-dev                                                                                           --------------------------------------------------------------------------------------                    https://github.com/wc-platform-engineering/terragrunt-onedrive-infra/blob/main/infrastructure/deployments/defaults.yaml              >>>>>>>>>>>>>>>>        ---
# Use this file for configuring variables that should apply to the all deployments in this repository.
# Most commonly this scope is used to set high level metadata properties.

# This project_name is used to namespace related resources together. When using our internal
# terraform naming module this value will be passed in as an input variable then applied
# as part of the name of each resource that gets created.
# If project_name is not explicitly set in any configuration then it will be set automatically
# using the name of the "project" directory the deployment runs under. It is recommended
# to explicitly set this to prevent unintended changes from occurring later if
# project structure changes.
# This variable is used for tagging.
project_name: projectdemo

# The project_owner variable is set to the friendly name of whomever is responsible for the
# resources deployed from this repository.
# This variable is used for tagging.
project_owner: Austin Quenneville

# The project_contact variable is set in addition to the project_owner to provide the email
# to contact for issues with resources deployed from this repository.
# This variable is used for tagging.
project_contact: austin.quenneville@whitecase.com

# The gitlab_project_id variable is only required when using GitLab as the Terraform state backend.
# When we move to another location, such as AzureRM Storage accounts this should be removed.
gitlab_project_id: 12345 # This value is a placeholder that does not correspond to an actual project

# The azure_tenant_id defines the Azure tenant that resources will be deployed to.
# This is the Azure Tenant ID that resources will be deployed to.
azure_tenant_id: b1fccc08-e170-4e18-a5d2-7a62fe61a082 # whitecase

# The azure_default_subscription_id variable is used to define a default subscription to
# be used when azure_subscription_id is not defined. Azure CLI and AzureRM terraform providers
# require a subscription ID to be set, even if the deployment does not involved subscription-scoped
# resources.
azure_default_subscription_id: 34b12d2d-e28c-43b5-a33b-12da8579925d # sub-plateng-dev-primary

# This variable is needed to replace the implicit "tenant" provided when using the larger
# directory structure used for mono-repo style repositories. We want to retain the same
# state name structure so we have to explicitly provide it so we don't lose the context.
tenant_name:                                                                                                            --------------------------------------------------------------------------------------------                 https://github.com/wc-platform-engineering/terragrunt-onedrive-infra/blob/main/infrastructure/deployments/root.hcl     >>>>>                                           # =====================================================================================
# Root Terragrunt Configuration
# This file provides base configurations that will be consumed by all child terragrunt
# modules within this project.
# =====================================================================================
locals {
  context_content = read_terragrunt_config("${get_repo_root()}/infrastructure/common/context.hcl")
  context         = local.context_content.locals
}

# This block defines the remote state that terraform will utilize when deploying resources.
# This project will just demonstrate with a local tfstate file, so there will be no actual
# backend configured for now. This should be updated to reflect using an AzureRM storage
# account but for a simple demo this should suffice.

## This is a configuration to show using a GitLab backend, but it's not needed in this example.
# remote_state {
#   backend = "http"
#   generate = {
#     path = "backend.tf"
#     if_exists = "overwrite_terragrunt"
#   }
#   config = {
#     address        = local.context.terraform_state_address
#     lock_address   = "${local.context.terraform_state_address}/lock"
#     unlock_address = "${local.context.terraform_state_address}/lock"
#     # The terraform_username and terraform_password values are to be loaded through the
#     # environment variables "TF_USERNAME" and "TF_PASSWORD"
#     lock_method    = "POST"
#     unlock_method  = "DELETE"
#     retry_wait_min = "5"
#   }
# }                                                                                                                                           ------------------------------------------------------------------------------------------      then the module folder : https://github.com/wc-platform-engineering/terragrunt-onedrive-infra/blob/main/infrastructure/modules/.KEEP  >>>>>> with empty .KEEP         file where i beleive we need to put all the modules . finally a readme file tells about every things about teh templet ..... >>>>>>>                                                                   Terragrunt Project Structure
Terragrunt is used to manage one or more Terraform/OpenTofu deployments within the same project. A Terraform/OpenTofu deployment, is the set of resources created by an apply operation that exist within a single state.

The directories here each have an explicit purpose that will be covered in more detail within each individual readme, but at a high level here is what you need to know:

modules: Contains any terraform root modules that are used by the deployments within this project.
common: Contains shared configurations used by one or more of the deployments within the project. Most of the HCL code will be found here.
deployments: Contains a verbose directory structure that terminates at each terragrunt.hcl which represents an individual deployment.
Terminology
Before deep diving into the structure of each one of these directories you should familiarize yourself with specific terminology used throughout our terragrunt projects so the different components comprising the project can be more easily understood. Some of these terms are related specifically to Azure, but where possible an alternative may provided for when this structure would apply to AWS.

Term	Definition	AWS Alternative
tenant	The Azure tenant to deploy to.	organization
subscription	The Azure subscription to deploy to.	account
project	A unique project, solution, or namespace to associate the resources with.	
environment	A deployent tier, or stage, such as prod, stg, or dev.	
location	The Azure location to deploy to.	region
unit	An individual Terraform/OpenTofu module to be deployed.	
deployment	A unique instance of a unit deployment. Such as blue and green.
The default to use for a single instance of a unit is primary.	
modules
The modules directory contains any root terraform modules used by any deployment within this project.

Terragrunt does not require a root module for deployment, so depending on the project this may be an empty or omitted directory if there are no unique root modules.

If root modules are necessary for a project, then each should be defined in it's own directory similar to the structure shown below:

modules/
├── network/
│   └── main.tf
└── application/
    └── main.tf
These modules are then available to be locally referenced for usage by deployments within the terragrunt.hcl or a shared .hcl file within ./common/.

For more information modules and examples of how they are used, please refer to the Modules README.

common
The common directory contains shared configurations that are used to provide the majority of configuration for one or more deployments within the project. These configurations mostly consist of .hcl files, but may also include other formats of configuration, such as .yaml as needed by the project.

Here you will also find the context.hcl that is used to merge variables from different scopes into a single source that can be used by any deployment.

The bulk of the .hcl configuration should be found within the shared configurations here rather than down with the narrowly scoped terragrunt.hcl files found within deployments. These configuration files keep the terragrunt.hcl files very lightweight, and makes it easier to update certain type of deployment and have the configuration change be reflected across all instances of it without having to modify each one individually.

deployments
The deployments directory contains a strict structure that is used to logically organization different deployments within a project and provide different scopes where configuration may be applied. Here is an example that demonstrates the structure used for a single deployment.

deployments/
└── wc/                               <-- tenant
    └── plateng-dev/                  <-- subscription
        └── platform/                 <-- project
            └── dev/                  <-- environment
                └── eastus/           <-- location
                    └── vnet/         <-- unit
                        └── primary/  <-- deployment
                            └── terragrunt.hcl
Each directory that terminates with a terragrunt.hcl file indicates a unique deployment for the project. The deployment will include any configuration found when traversing up the tree back to the root /deployments/ directory.

Terraform states are directly named in reference to this structure. If we were to deploy the example above it would result in a state named:

wc-plateng_dev-platform-dev-eastus-vnet-primary

Expanding on the example shown above, if we wanted to add an additional deployment of our vnet in a different location, the structure would look like this:

deployments/
└── wc/                               <-- tenant
    └── plateng-dev/                  <-- subscription
        └── platform/                 <-- project
            └── dev/                  <-- environment
                ├── eastus/           <-- location
                │   └── vnet/         <-- unit
                │       └── primary/  <-- deployment
                │           └── terragrunt.hcl
                └── swedencentral/    <-- location
                    └── vnet/         <-- unit
                        └── primary/  <-- deployment
                            └── terragrunt.hcl
Contributing and Feedback
This structure is intended to be iterated on to best serve our needs. If you have any suggestions about how to improve any of this structure, how we use terragrunt, or anything else that would contribute to making deployments easier or configurations less complex please feel free to submit a pull request or speak with the team about those ideas.

As a note, this structure was designed based on earlier iterations of terragrunt so there may be more modern approaches that we could look to move towards, such as the blueprint based terragrunt stacks with terragrunt.stack.hcl files. The absence of some of these more modern approaches does not mean that they have been ruled out, more likely we just haven't had the time to invest into implementing them. ,,,,, so that is all start working on this solution please ,,,,, with explenations and step by step to implement the  solution . go Ashly                                                                                                  



,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,, that it is all,,,, so now let work and focuse on the terragrunt step conversion ,,,, i need explanation and step by step of what to do and a full repo changes in the code to with the fiall full code ready to copy and paste ,,, need u to do the whole repo and bout the last extra dev paths u can apply the sam eon staging and prod ,,,, dont delete or change any thing without adding a comment ,,, go a head do it again from the begging.... lets goooooooo ashly ;)
