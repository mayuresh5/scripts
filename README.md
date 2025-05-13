# scripts

#!/bin/bash

Output file
OUTPUT_FILE="idle_cluster_recommendations.json"
echo "[]" > "$OUTPUT_FILE"

Get all project IDs (excluding deleted ones)
PROJECT_IDS=$(gcloud projects list --format="value(projectId)")

echo "Scanning projects for 'Delete idle cluster' recommendations..."

for PROJECT_ID in $PROJECT_IDS; do
echo "Checking project: $PROJECT_ID"

bash
Copy
Edit
# Check if recommendation API is enabled
API_STATUS=$(gcloud services list \
    --project="$PROJECT_ID" \
    --filter="recommendationengine.googleapis.com" \
    --format="value(config.name)")

if [[ -z "$API_STATUS" ]]; then
    echo " - Recommendation API not enabled. Skipping."
    continue
fi

# Get recommendations
RECOMMENDATIONS=$(gcloud recommender recommendations list \
    --project="$PROJECT_ID" \
    --location=global \
    --recommender=google.cloud.gkeHub.clusterIdleRecommender \
    --format=json 2>/dev/null)

if [[ $? -ne 0 || "$RECOMMENDATIONS" == "[]" ]]; then
    echo " - No idle cluster recommendations found. Skipping."
    continue
fi

# Filter only category COST & recommendation DELETE_IDLE_CLUSTER
MATCHING=$(echo "$RECOMMENDATIONS" | jq '[.[] | select(.recommenderSubtype == "DELETE_IDLE_CLUSTER" and .category == "COST")]')

if [[ "$MATCHING" == "[]" ]]; then
    echo " - No matching recommendations. Skipping."
    continue
fi

echo " - Found recommendations. Adding to output."

# Append to output file
jq -s '.[0] + .[1]' "$OUTPUT_FILE" <(echo "$MATCHING") > tmp.json && mv tmp.json "$OUTPUT_FILE"
done

echo "✅ Done. Results stored in: $OUTPUT_FILE"
