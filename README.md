#!/bin/bash

# Script to fetch idle cluster recommendations across multiple GCP projects
# and save the output to a text file

# Set the location and other parameters
LOCATION="asia-south1"
RECOMMENDER="google.container.DiagnosisRecommender"
FILTER="recommenderSubtype=CLUSTER_IDLE"

# Output file name with timestamp
OUTPUT_FILE="idle_clusters_$(date +%Y%m%d_%H%M%S).txt"

# Check if projects are provided
if [ $# -eq 0 ]; then
    echo "No projects specified. Please provide project IDs as arguments."
    echo "Usage: $0 project1 project2 ..."
    exit 1
fi

echo "====== Idle Cluster Recommendations ======" | tee "$OUTPUT_FILE"
echo "Started at: $(date)" | tee -a "$OUTPUT_FILE"
echo "" | tee -a "$OUTPUT_FILE"

# Loop through each project provided as an argument
for PROJECT in "$@"; do
    echo "===== Processing project: $PROJECT =====" | tee -a "$OUTPUT_FILE"
    
    # Run the gcloud command, show output on CLI and append to file
    gcloud recommender recommendations list \
      --project="$PROJECT" \
      --location="$LOCATION" \
      --recommender="$RECOMMENDER" \
      --filter="$FILTER" | tee -a "$OUTPUT_FILE"
    
    # Add a separator between projects
    echo "" | tee -a "$OUTPUT_FILE"
    echo "---------------------------------------" | tee -a "$OUTPUT_FILE"
    echo "" | tee -a "$OUTPUT_FILE"
done

echo "Completed at: $(date)" | tee -a "$OUTPUT_FILE"
echo "Results saved to: $OUTPUT_FILE"
