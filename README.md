# AppServicesBestPractices

_subscriptionId=""

BOLD=$(tput bold)
RESET=$(tput sgr0)
RED='\033[0;31m'
BRIGHTRED='\e[91m'
GREEN='\033[0;32m'
YELLOW='\033[0;33m'
BORDER="\e[47m\e[30m"

# Define color codes for green, red, and reset
color_enabled="\e[32m"    # Green for "ENABLED"
color_disabled="\e[93;1m" # Red for "DISABLED"
color_reset="\e[0m"       # Reset to default color

echo -e "${BORDER}${YELLOW}${BOLD}*****************************************************************************************************************************${RESET}"
echo -e "${BORDER}${GREEN}${BOLD}------------------------------------------ APP SERVICES RECOMMENDATIONS -------------------------------------------------------${RESET}"
echo -e "${BORDER}${YELLOW}${BOLD}*****************************************************************************************************************************${RESET}"
printf "\n\n"
echo -e "App Services Common Best Practices Configuration"
printf "\n\n"
printf "%-50s %-25s %-25s %-25s\n" "APP SERVICE NAME" "AUTO HEAL" "HEALTH CHECK" "ALWAYS ON"
printf "%s\n" "-----------------------------------------------------------------------------------------------------------------"

_diplayAppServiceRecommendations_common=0

_webApps=$(az webapp list --subscription $_subscriptionId --query "[].{name:name, resourceGroup:resourceGroup}" --output jsonc)
for item in $(echo "$_webApps" | jq -r '.[] | @base64'); do
    _jq() {
        echo ${item} | base64 --decode | jq -r ${1}
    }
    _name=$(_jq '.name')
    _resourceGroup=$(_jq '.resourceGroup')
    _autoHealEnabled=$(az webapp config show --subscription $_subscriptionId --resource-group $_resourceGroup --name $_name --query "autoHealEnabled" --output tsv | grep -q "^true$" && echo "ENABLED" || echo "DISABLED")
    _healthCheckPath=$(az webapp config show --subscription $_subscriptionId --resource-group $_resourceGroup --name $_name --query "healthCheckPath" --output tsv | grep -q . && echo "ENABLED" || echo "DISABLED")
    _alwaysOn=$(az webapp config show --subscription $_subscriptionId --resource-group $_resourceGroup --name $_name --query "alwaysOn" --output tsv | grep -q "^true$" && echo "ENABLED" || echo "DISABLED")

    # Apply colors conditionally for each field
    if [[ "$_autoHealEnabled" == "ENABLED" ]]; then
        autoHealColor=$color_enabled
    elif [[ "$_autoHealEnabled" == "DISABLED" ]]; then
        autoHealColor=$color_disabled
    else
        autoHealColor=$color_reset
    fi

    if [[ "$_healthCheckPath" == "ENABLED" ]]; then
        healthCheckColor=$color_enabled
    elif [[ "$_healthCheckPath" == "DISABLED" ]]; then
        healthCheckColor=$color_disabled
    else
        healthCheckColor=$color_reset
    fi

    if [[ "$_alwaysOn" == "ENABLED" ]]; then
        alwaysOnColor=$color_enabled
    elif [[ "$_alwaysOn" == "DISABLED" ]]; then
        alwaysOnColor=$color_disabled
    else
        alwaysOnColor=$color_reset
    fi

    printf "\e[34m%-50s\e[0m ${autoHealColor}%-25s\e[0m ${healthCheckColor}%-25s\e[0m ${alwaysOnColor}%-25s\e[0m\n" "$_name" "$_autoHealEnabled" "$_healthCheckPath" "$_alwaysOn"

done

##############################################################################
## App Service Plan
##############################################################################

printf "\n\n"

_diplayAppServicePlanRecommendations_density=0
_diplayAppServicePlanRecommendations_cost=0

app_service_plans=$(az appservice plan list --subscription $_subscriptionId --query "[].{Name:name, ResourceGroup:resourceGroup}" --output tsv)
while IFS=$'\t' read -r name resource_group; do

    webapp_count=$(az webapp list --subscription $_subscriptionId --query "[?appServicePlanId=='/subscriptions/$_subscriptionId/resourceGroups/$resource_group/providers/Microsoft.Web/serverfarms/$name'] | length(@)" -o tsv)

    webapp_info=$(az appservice plan show --name $name --subscription $_subscriptionId --resource-group $resource_group --query "{tier:sku.tier, size:sku.name, workers:sku.capacity}" --output json)
    webapp_worker=$(echo "$webapp_info" | jq -r '.workers')
    webapp_size=$(echo "$webapp_info" | jq -r '.size')
    webapp_tier=$(echo "$webapp_info" | jq -r '.tier')

    if [ "$_diplayAppServicePlanRecommendations_density" -eq 0 ]; then
        echo -e "${BOLD}${YELLOW}APP SERVICE PLAN DENSITY CHECK ---------------------------------------------------------------------------${RESET}"
        _diplayAppServicePlanRecommendations_density=1
        printf "\n\n"
        # Print table header
        printf "%-50s %-25s %-25s %-25s %-25s\n" "APP SERVICE PLAN" "SIZE" "TIER" "RECOMMENDED APPS COUNT" "CURRENT APPS COUNT"
        printf "%s\n" "----------------------------------------------------------------------------------------------------------------------------------------------------"
    fi

    # Check if webapp_count is greater than 1
    if [ "$webapp_count" -gt 1 ]; then
        HighCount=$YELLOW # Set color to bright red if count is greater than 1
    else
        HighCount=$color_reset # Reset color if count is 1 or less
    fi

    if [[ "$webapp_size" =~ ^(B1|S1|P1v2|I1v1|P0v3|P1)$ ]]; then
        printf "%-50s %-25s %-25s %-25s ${HighCount}%-25s\e[0m\n" "$name" "$webapp_size" "$webapp_tier" "8" "$webapp_count"
    elif [[ "$webapp_size" =~ ^(B2|S2|P2v2|I2v1|P2)$ ]]; then
        printf "%-50s %-25s %-25s %-25s ${HighCount}%-25s\e[0m\n" "$name" "$webapp_size" "$webapp_tier" "16" "$webapp_count"
    elif [[ "$webapp_size" =~ ^(B3|S3|P3v2|I3v1|P3)$ ]]; then
        printf "%-50s %-25s %-25s %-25s ${HighCount}%-25s\e[0m\n" "$name" "$webapp_size" "$webapp_tier" "32" "$webapp_count"
    elif [[ "$webapp_size" =~ ^(P1v3|I1v2)$ ]]; then
        printf "%-50s %-25s %-25s %-25s ${HighCount}%-25s\e[0m\n" "$name" "$webapp_size" "$webapp_tier" "16" "$webapp_count"
    elif [[ "$webapp_size" =~ ^(P2v3|I2v2)$ ]]; then
        printf "%-50s %-25s %-25s %-25s ${HighCount}%-25s\e[0m\n" "$name" "$webapp_size" "$webapp_tier" "32" "$webapp_count"
    elif [[ "$webapp_size" =~ ^(P3v3|I3v2)$ ]]; then
        printf "%-50s %-25s %-25s %-25s ${HighCount}%-25s\e[0m\n" "$name" "$webapp_size" "$webapp_tier" "64" "$webapp_count"
    fi

    ## COST RECOMMENDATIONS

    if [ "$webapp_count" -eq 0 ]; then

        if [ "$_diplayAppServicePlanRecommendations_cost" -eq 0 ]; then
            printf "\n\n"
            echo -e "${BOLD}${YELLOW} APP SERVICE PLAN RECOMMENDATIOONS COST---------------------------------------------------------------------------${RESET}}"
            _diplayAppServicePlanRecommendations_cost=1
            printf "\n\n"
            echo -e "${BOLD}${RED} FOLLOWING APP SERVICE PLAN DOES NOT HAVE ANY APP SERVICE, REIVEW AND DELETE TO OPTIMIZE THE COST${RESET}"
            printf "\n\n"
            printf "%-50s %-25s\n" "APP SERVICE PLAN" "App Count"
            printf "%s\n" "-----------------------------------------------------------------------------"
        fi
        printf "%-50s %-25s\n" "$name" "0"
    fi

done <<<"$app_service_plans"

printf "\n"

echo -e "${BORDER}${YELLOW}${BOLD}*****************************************************************************************************************************${RESET}"
echo -e "${BORDER}${GREEN}${BOLD}---------------------------------- END OF RECOMMENDATIONS - { MORE TO COME } -------------------------------------------------${RESET}"
echo -e "${BORDER}${YELLOW}${BOLD}*****************************************************************************************************************************${RESET}"

