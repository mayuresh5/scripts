#!/bin/bash

# Script to run recommender commands against all GCP projects
# Default location (can be overridden with -l or --location flag)
LOCATION="global"

# Function to display usage information
usage() {
  echo "Usage: $0 [OPTIONS]"
  echo "Options:"
  echo "  -l, --location LOCATION    Specify the location (default: global)"
  echo "  -a, --all-locations        Check multiple locations (global, regions)"
  echo "  -h, --help                 Display this help message"
  exit 1
}

# Parse command line arguments
CHECK_ALL_LOCATIONS=false
while [[ $# -gt 0 ]]; do
  case "$1" in
    -l|--location)
      LOCATION="$2"
      shift 2
      ;;
    -a|--all-locations)
      CHECK_ALL_LOCATIONS=true
      shift
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
echo "Location: $LOCATION (${CHECK_ALL_LOCATIONS:+checking multiple locations})"
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
    
    # Initialize variables
    FOUND_RECOMMENDATIONS=false
    BEST_LOCATION=""
    BEST_COUNT=0
    
    # Define locations to check
    declare -a LOCATIONS_TO_CHECK=("$LOCATION")
    
    # If check all locations flag is set, add more locations to check
    if [ "$CHECK_ALL_LOCATIONS" = true ]; then
      LOCATIONS_TO_CHECK=("global" "us-central1" "asia-south1" "europe-west1")
    fi
    
    # Try each location
    for LOC in "${LOCATIONS_TO_CHECK[@]}"; do
      TEMP_FILE=$(mktemp)
      
      echo -n "(Trying $LOC) ... "
      gcloud recommender recommendations list \
        --project="$PROJECT" \
        --location="$LOC" \
        --recommender="$RECOMMENDER" \
        --format=json > "$TEMP_FILE" 2>/dev/null
      
      CUR_EXIT_CODE=$?
      
      # If successful and contains data
      if [ $CUR_EXIT_CODE -eq 0 ] && [ -s "$TEMP_FILE" ]; then
        # Count recommendations
        CUR_COUNT=$(jq 'length' "$TEMP_FILE" 2>/dev/null)
        if [ -z "$CUR_COUNT" ] || [ "$CUR_COUNT" = "null" ]; then
          CUR_COUNT=0
        fi
        
        # If we found recommendations and it's more than before, save this as the best result
        if [ "$CUR_COUNT" -gt 0 ] && [ "$CUR_COUNT" -gt "$BEST_COUNT" ]; then
          FOUND_RECOMMENDATIONS=true
          BEST_COUNT=$CUR_COUNT
          BEST_LOCATION=$LOC
          cat "$TEMP_FILE" > "$OUTPUT_FILE"
        fi
        
        echo -n "($CUR_COUNT found) "
      fi
      
      rm -f "$TEMP_FILE"
      
      # If we're not checking all locations and found recommendations, break the loop
      if [ "$CHECK_ALL_LOCATIONS" = false ] && [ "$FOUND_RECOMMENDATIONS" = true ]; then
        break
      fi
    done
    
    # If we found recommendations in any location
    if [ "$FOUND_RECOMMENDATIONS" = true ]; then
      echo "SUCCESS (Best: $BEST_COUNT in $BEST_LOCATION)"
      REC_COUNT=$BEST_COUNT
      STATUS="SUCCESS"
    else
      # Try once more with the original location for error reporting
      gcloud recommender recommendations list \
        --project="$PROJECT" \
        --location="$LOCATION" \
        --recommender="$RECOMMENDER" \
        --format=json > "$OUTPUT_FILE" 2>/dev/null
      
      EXIT_CODE=$?
      STATUS=$(check_error $EXIT_CODE "$PROJECT" "$RECOMMENDER")
      echo "$STATUS"
      
      # Remove empty file if command failed
      rm -f "$OUTPUT_FILE"
      REC_COUNT="0"
    fi
    echo "$STATUS"
    
    # Count recommendations if successful
    if [ "$FOUND_RECOMMENDATIONS" = true ]; then
      # We already have REC_COUNT from above
      echo "    Found $REC_COUNT recommendation(s) in $BEST_LOCATION"
    else
      if [ "$REC_COUNT" = "0" ]; then
        echo "    No recommendations found in any checked location"
      fi
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
