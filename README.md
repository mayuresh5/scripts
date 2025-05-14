# scripts

#!/bin/bash

# Script to run recommender commands against all GCP projects
# Default location (can be overridden with -l or --location flag)
LOCATION="asia-south1"

# Function to display usage information
usage() {
  echo "Usage: $0 [OPTIONS]"
  echo "Options:"
  echo "  -l, --location LOCATION    Specify the location (default: asia-south1)"
  echo "  -h, --help                 Display this help message"
  exit 1
}

# Parse command line arguments
while [[ $# -gt 0 ]]; do
  case "$1" in
    -l|--location)
      LOCATION="$2"
      shift 2
      ;;
    -h|--help)
      usage
      ;;
    *)
      echo "Unknown option: $1"
      usage
      ;;
  esac
done

# Define all recommenders
declare -a RECOMMENDERS=(
  "google.container.Diagnosis.Recommender" 
  "google.compute.instance.IdleResourceRecommender"
  "google.compute.disk.IdleResourceRecommender"
  "google.compute.address.IdleResourceRecommender"
  "google.compute.image.IdleResourceRecommender"
  "google.compute.instance.MachineTypeRecommender"
  "google.compute.instanceGroupManager.MachineTypeRecommender"
  "google.cloudsql.instance.IdleRecommender"
  "google.cloudsql.instance.OverprovisionedRecommender"
  "google.resourcemanager.projectUtilization.Recommender"
  "google.billing.commitment.SpendBasedCommitmentRecommender"
  "google.compute.commitment.UsageCommitmentRecommender"
  "google.cloudrun.service.CostRecommender"
)

echo "========================================================"
echo "GCP Recommender Check Script"
echo "Location: $LOCATION"
echo "Started at: $(date)"
echo "========================================================"

# Get list of all accessible projects
echo "Fetching list of accessible projects..."
PROJECTS=$(gcloud projects list --format="value(projectId)")

if [ -z "$PROJECTS" ]; then
  echo "Error: No projects found or accessible. Please check your gcloud authentication."
  exit 1
fi

# Create a directory for output
OUTPUT_DIR="recommender_output_$(date +%Y%m%d_%H%M%S)"
mkdir -p "$OUTPUT_DIR"
echo "Output will be saved to directory: $OUTPUT_DIR"

# Initialize summary file
SUMMARY_FILE="$OUTPUT_DIR/summary.txt"
echo "Project,Recommender,Recommendations Count,Status" > "$SUMMARY_FILE"

# Function to check if a command failed
check_error() {
  local exit_code=$1
  local project=$2
  local recommender=$3
  
  if [ $exit_code -eq 0 ]; then
    echo "SUCCESS"
    return 0
  else
    # Different error codes may indicate different issues
    case $exit_code in
      4)
        echo "NOT_APPLICABLE (This recommender not applicable for this project)"
        ;;
      *)
        echo "FAILED (Error code: $exit_code)"
        ;;
    esac
    return 1
  fi
}

# Count of projects processed
TOTAL_PROJECTS=$(echo "$PROJECTS" | wc -l)
CURRENT_PROJECT=0

# Process each project
for PROJECT in $PROJECTS; do
  CURRENT_PROJECT=$((CURRENT_PROJECT + 1))
  echo ""
  echo "========================================================"
  echo "Processing project $CURRENT_PROJECT of $TOTAL_PROJECTS: $PROJECT"
  echo "========================================================"
  
  # Create a project-specific directory
  PROJECT_DIR="$OUTPUT_DIR/$PROJECT"
  mkdir -p "$PROJECT_DIR"
  
  # Process each recommender for this project
  for RECOMMENDER in "${RECOMMENDERS[@]}"; do
    echo -n "  Checking recommender: $RECOMMENDER ... "
    
    # Define output file for this recommender
    OUTPUT_FILE="$PROJECT_DIR/$(echo $RECOMMENDER | tr '.' '_').json"
    
    # Run the gcloud command and capture the output and error code
    gcloud recommender recommendations list \
      --project="$PROJECT" \
      --location="$LOCATION" \
      --recommender="$RECOMMENDER" \
      --format=json > "$OUTPUT_FILE" 2>/dev/null
    
    EXIT_CODE=$?
    STATUS=$(check_error $EXIT_CODE "$PROJECT" "$RECOMMENDER")
    echo "$STATUS"
    
    # Count recommendations if successful
    if [ $EXIT_CODE -eq 0 ]; then
      # Check if the output is a valid JSON array and count the items
      if [ -s "$OUTPUT_FILE" ]; then
        REC_COUNT=$(jq 'length' "$OUTPUT_FILE")
        if [ -z "$REC_COUNT" ]; then
          REC_COUNT=0
        fi
      else
        REC_COUNT=0
      fi
      echo "    Found $REC_COUNT recommendation(s)"
    else
      REC_COUNT="N/A"
      # Remove empty file if command failed
      rm -f "$OUTPUT_FILE"
    fi
    
    # Update summary file
    echo "$PROJECT,$RECOMMENDER,$REC_COUNT,$STATUS" >> "$SUMMARY_FILE"
  done
done

echo ""
echo "========================================================"
echo "Execution completed at: $(date)"
echo "Results saved in: $OUTPUT_DIR"
echo "Summary file: $SUMMARY_FILE"
echo "========================================================"

# Display a summary of recommendations found
echo "Recommendations Summary:"
echo "----------------------"
awk -F ',' 'NR>1 && $3 != "N/A" && $3 > 0 {sum[$2]+=$3; projects[$2]=projects[$2]","$1} END {for (r in sum) {printf "%-60s: %5d recommendations across %d projects\n", r, sum[r], split(projects[r],dummy,",")-1}}' "$SUMMARY_FILE" | sort -k3,3nr
