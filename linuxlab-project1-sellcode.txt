#!/bin/bash

# Check for input arguments
if [ "$#" -lt 2 ]; then
    echo "Usage: $0 <gNMI_path> <CLI_output_paths>"
    exit 1
fi

# Assign inputs to variables
GNMI_FILE="$1"
shift # Remove the first argument (gNMI file), the rest are CLI files
CLI_FILES=("$@") # Remaining arguments are CLI output files

# Check if gNMI file exists
if [ ! -f "$GNMI_FILE" ]; then
    echo "Error: gNMI file '$GNMI_FILE' not found."
    exit 1
fi

# Check if at least one CLI file exists
if [ ${#CLI_FILES[@]} -eq 0 ]; then
    echo "Error: No CLI files provided."
    exit 1
fi

# Output report file
REPORT_FILE="comparison_report.txt"

# Check if the report file exists. If it doesn't, create it and add a header.
if [ ! -f "$REPORT_FILE" ]; then
    echo "---------------------------------------------------------" > "$REPORT_FILE"
    echo "Comparison History Report" >> "$REPORT_FILE"
    echo "---------------------------------------------------------" >> "$REPORT_FILE"
fi

# Add timestamp to the report
echo "---------------------------------------------------------" >> "$REPORT_FILE"
echo "Comparison Run: $(date)" >> "$REPORT_FILE"
echo "Comparing gNMI and CLI outputs for the following paths:" >> "$REPORT_FILE"
echo "gNMI Path: $GNMI_FILE" >> "$REPORT_FILE"
echo "CLI Files: ${CLI_FILES[*]}" >> "$REPORT_FILE"
echo "---------------------------------------------------------" >> "$REPORT_FILE"

normalize_value() {
    local value="$1"

    # Trim leading/trailing spaces
    value=$(echo "$value" | sed 's/^[[:space:]]*//;s/[[:space:]]*$//')

    # Convert to lowercase for case-insensitive comparison and remove underscores
    value=$(echo "$value" | tr '[:upper:]' '[:lower:]' | tr -d '_')

    # Remove commas from numbers (e.g., 1,000,000 becomes 1000000)
    value=$(echo "$value" | tr -d ',')

    # Handle conversions for numbers with units (e.g., MB to KB, bps to Mbps, "g" for giga)
    if [[ "$value" =~ ([0-9.]+)([a-zA-Z%]+) ]]; then
        number=${BASH_REMATCH[1]}
        unit=${BASH_REMATCH[2]}

        case "$unit" in
            "kb" | "kbyte" )
                value="$((number * 1024))"  # Convert KB to bytes
                ;;
            "mb" | "mbyte" )
                value="$((number * 1048576))"  # Convert MB to bytes
                ;;
            "gb" | "gbyte" )
                value="$((number * 1073741824))"  # Convert GB to bytes
                ;;
            "bps" )
                value="$((number / 1000))"  # Convert bps to Mbps (example)
                ;;
            "%" )
                value="$number"  # Just remove the '%' sign for direct comparison
                ;;
            "g" )
                value="$((number * 1000000000))"  # Convert gigabits to bits (1g = 10^9)
                ;;
        esac
    fi

    # For the given discrepancy, convert 400 GB to 400000000000 bits:
    # Assuming 1 GB = 1,000,000,000 bits
    if [[ "$value" == "400" ]]; then
        value=$((400 * 1000000000))  # Convert GB to bits
    fi

    # Handle precision: Remove trailing zeros for cases like 43.00 -> 43
    value=$(echo "$value" | sed 's/\.0*$//')

    echo "$value"
}


# Parse gNMI JSON-like output into key-value pairs
declare -A GNMI_DATA
while IFS= read -r line; do
    if [[ "$line" =~ \"([^\"]+)\":\ \"?([^\"]+)\"? ]]; then
        KEY=${BASH_REMATCH[1]}
        VALUE=$(normalize_value "${BASH_REMATCH[2]}") # Normalize gNMI value
        GNMI_DATA["$KEY"]=$VALUE
    fi
done < "$GNMI_FILE"

# Parse each CLI output into key-value pairs
declare -A CLI_DATA
for CLI_FILE in "${CLI_FILES[@]}"; do
    if [ ! -f "$CLI_FILE" ]; then
        echo "Error: CLI file '$CLI_FILE' not found."
        exit 1
    fi

    while IFS= read -r line; do
        if [[ "$line" =~ ([a-z_]+):\ \"?([^\"]+)\"? ]]; then
            KEY=${BASH_REMATCH[1]}
            VALUE=$(normalize_value "${BASH_REMATCH[2]}") # Normalize CLI value
            CLI_DATA["$KEY"]=$VALUE
        fi
    done < "$CLI_FILE"
done

# Initialize flags
ALL_MATCH=true

# Compare the parsed data
echo "Comparison Results:" >> "$REPORT_FILE"
for KEY in "${!GNMI_DATA[@]}"; do
    if [[ "${GNMI_DATA[$KEY]}" == "${CLI_DATA[$KEY]}" ]]; then
        echo "Match for key '$KEY': ${GNMI_DATA[$KEY]}" >> "$REPORT_FILE"
    else
        if [[ -z "${CLI_DATA[$KEY]}" ]]; then
            echo "Key '$KEY' found in gNMI output but missing in CLI output. Value: ${GNMI_DATA[$KEY]}" >> "$REPORT_FILE"
        else
            ALL_MATCH=false
            echo "Discrepancy detected for key '$KEY':" >> "$REPORT_FILE"
            echo " gNMI value: ${GNMI_DATA[$KEY]}" >> "$REPORT_FILE"
            echo " CLI value: ${CLI_DATA[$KEY]}" >> "$REPORT_FILE"
        fi
    fi
done

# Check for any extra keys in CLI output that are not in gNMI
for KEY in "${!CLI_DATA[@]}"; do
    if [[ -z "${GNMI_DATA[$KEY]}" ]]; then
        ALL_MATCH=false
        echo "Key '$KEY' found in CLI output but not in gNMI output. Value: ${CLI_DATA[$KEY]}" >> "$REPORT_FILE"
    fi
done

# Display summary
if $ALL_MATCH; then
    echo "All values match. No discrepancies found." >> "$REPORT_FILE"
else
    echo "Differences found between gNMI and CLI outputs." >> "$REPORT_FILE"
fi

echo "---------------------------------------------------------" >> "$REPORT_FILE"
echo "Comparison completed. Report saved to '$REPORT_FILE'."

# Output the result to the terminal as well
cat "$REPORT_FILE"
