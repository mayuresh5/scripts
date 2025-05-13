# scripts

#!/bin/bash

# Script to fetch "Delete idle cluster" recommendations from GCP projects
# Only fetches Cost category recommendations

# Set output file
OUTPUT_FILE="idle_cluster_recommendations.csv"

# Create CSV header
echo "Project ID,Recommendation ID,Description,Last Refresh Time,Primary Impact,State" > "$OUTPUT_FILE"

# Get list of all projects
projects=$(gcloud projects list --format="value(projectId)")

# Counter for logging
total_projects=$(echo "$projects" | wc -l)
current=0

# Function to check if recommender API is enabled
check_api_enabled() {
    local project_id=$1
    gcloud services list --project="$project_id" | grep -q recommender.googleapis.com
    return $?
}

# Function to enable recommender API
enable_api() {
    local project_id=$1
    echo "Recommender API not enabled in project $project_id. Skipping..."
    # Uncomment the line below if you want to enable the API automatically
    # gcloud services enable recommender.googleapis.com --project="$project_id"
}

# Loop through each project
for project_id in $projects; do
    current=$((current+1))
    echo "Processing project $current/$total_projects: $project_id"
    
    # Check if recommender API is enabled
    if ! check_api_enabled "$project_id"; then
        enable_api "$project_id"
        continue
    fi
    
    # Get cost recommender ID for idle clusters
    # Note: The exact recommender ID might vary - this is for Dataproc idle clusters
    recommender_id="google.dataproc.cluster.ClusterResourceIdle"
    
    # Try to fetch recommendations
    recommendations=$(gcloud recommender recommendations list \
        --project="$project_id" \
        --location=global \
        --recommender="$recommender_id" \
        --format="json" 2>/dev/null)
    
    # Check if we got any recommendations
    if [ -z "$recommendations" ] || [ "$recommendations" == "[]" ]; then
        echo "No idle cluster recommendations found for project $project_id"
        continue
    fi
    
    # Process recommendations
    echo "$recommendations" | jq -r '.[] | select(.recommenderSubtype=="DELETE_RESOURCE") | [
        "'"$project_id"'",
        .name,
        .description,
        .lastRefreshTime,
        .primaryImpact.category,
        .stateInfo.state
    ] | @csv' >> "$OUTPUT_FILE"
    
    echo "Found recommendations for project $project_id"
done

echo "Script completed. Results saved to $OUTPUT_FILE"
