# bash-script-that-primarily-downloads-PDF-files-from-a-specified-URL

#!/bin/bash

# Name: Manito Francine Beredo
# Student number: 10632689

# I wrote a bash script that primarily downloads PDF files from a specified URL. Throughout the script,I used five functions. 
# The first function is "is_valid_url," which takes the URL as an input and checks the URL's validity using the command "curl".
# The second function is the "download_pdfs". It calls the "is_valid_url" to check if the URL is valid. If it is invalid, an 
# error message will appear and the script will exit. If the URL is correct, the "date" command will automatically construct a timestamp 
# to store the downloaded PDF files. The "wget" command was used in the function to download HTML content from a URL.
# The "awk" command was then used to extract the PDF files. If there is no PDF at the specified URL, an error message will be produced  
# and the script will exit. The third function is "display_pdf_info," which uses the "wget" command to recover HTML text and "grep" to retrieve PDF links. 
# When executed, it will output the file name, size in bytes, and size label (b, Kb, or Mb). "create_zip" is the fourth function. 
# This function creates a zip archive of the files using "zip" command. Lastly, the "main" function is the script's entry point.



# Function to check if a URL is valid
function is_valid_url() {
    local url=$1
    if curl --output /dev/null --silent --head --fail "$url"; then
        return 0
    else
        return 1
    fi
}

# Function to download PDF files from a URL
function download_pdfs() {
    local url=$1

    # Check if the URL that is given by user is valid
    if ! is_valid_url "$url"; then
        echo "URL is invalid. Please provide a valid URL."      # Print invalid message
        exit 1
    fi

    # Create timestamp for the directory name
    timestamp=$(date +%Y_%m_%d_%H_%M_%S)
    directory="pdfs_$timestamp"
    mkdir "$directory"  # Create directory

    html_content=$(wget -qO- "$url")        # Download the HTML content from URL
    pdf_files=$(echo "$html_content" | awk -F'"' '/href.*\.pdf/{print $2}')     # Extract PDF file URLs from the HTML content

    # Check if there is any PDF files found
    if [[ -z "$pdf_files" ]]; then
        echo -e "\e[31mNo PDFs found at this URL - exiting..\e[0m"      # Print in red
        exit 0
    fi

    count=0     # A variable to keep track of the number of downloaded files

    for pdf_file in $pdf_files; do

        # Checks if file with the same name exisis in specified directory
        if [[ -f "$directory/$pdf_file" ]]; then
            echo "Skipping duplicate file: $pdf_file"       # Prints message
            continue
        fi

        wget -q "$url/$pdf_file" -P "$directory"    # Download PDF file from URL and save to directory. '-q' option is used to surpress output of 'wget' command
        ((count++))     # For each successfully downloaded file, this line adds one to the count variable.
    done

    # Checks if no new files were downloaded
    if [[ $count -eq 0 ]]; then
        echo "No new PDF files downloaded."     # Print message
        exit 0
    fi

    echo -e "Downloading . . . . . . \e[32m$count\e[0m PDF files have been downloaded to the directory \e[32m$directory\e[0m"       # Summary message in green
}

# Function to display information about downloaded PDF files
function display_pdf_info() {
    local url=$1    # Parameter

    wget_output=$(wget -qO- "$url")     # Retrieve the HTML content
    pdf_links=$(echo "$wget_output" | grep -oP '<a[^>]+href="[^\"]+\.pdf"')     # Extract PDF file links. The "grep" command searches for lines in the input that match the given pattern

    echo -e "\e[34mFile Name\e[0m         \e[34mSize(Bytes)\e[0m"   # Prints header, displaying 'file name' and 'bytes'

    while IFS= read -r line; do     # Start of loop which reads each line
        file_url=$(echo "$line" | grep -oP 'href="\K[^"]+')     # 'grep' is used to extract URL from the lines
        file_name=$(basename "$file_url")       # 'basname' used to extract file name from URL
        file_size=$(curl -sI "$file_url" | awk '/Content-Length/ {print $2}' | tr -d '\r')      # 'curl' used to retrieve file size 'awk' command is used to extract content length from response headers

    # Checks file size
    if (( file_size >= 1048576 )); then     # If size greater than (1024*1024)
        file_size=$(echo "scale=2; $file_size / 1048576" | bc)  # Divided by 1048576 convert to megabytes
        file_size_label="Mb"    # Convert to megabytes
    elif (( file_size >= 1024 )); then      # If size between 1024 and 1048576
        file_size=$(echo "scale=2; $file_size / 1024" | bc)     # Divided by 1024
        file_size_label="Kb"    # Convert to kilobytes
    else
        file_size_label="b"     # If less than 1024, kept as bytes
    fi
        echo -e "$file_name  \e[34m|\e[0m  $file_size $file_size_label"     # Prints each file name,size and size label
    done <<< "$pdf_links"
}

# Function to create a zip archive of downloaded PDF files
function create_zip() {
    local directory=$1
    zip_file="$directory.zip"   # Name of the zip file that will be created
    zip -q -r "$zip_file" "$directory"  # The 'q' option surpress the output of zip command, and 'r' option includes all files and directories
    echo -e "PDFs archived to \e[32m$zip_file\e[0m in the \e[32m$directory\e[0m directory."     # Print message that informs user that the PDFs are archived
}

# Main script
function main() {
    if [[ $1 == "-z" ]]; then       # Check if the command-line flag is "-z"
        read -p "Enter a URL: " url     # Prompt user to enter URL

        download_pdfs "$url"    # Call download_pdfs function and pass URL as an argument
        display_pdf_info "$url"     # Call display_pdf_info function and pass URL as an argument
        create_zip "$directory"     # Call create_zip function and pass the directory as an argument
    
    elif [[ $1 == "" ]]; then   # Check if it is a command-line argument is an empty string
        read -p "Enter a URL: " url

        download_pdfs "$url"    # Call download_pdfs function and pass URL as an argument
        display_pdf_info "$url"     # Call display_pdf_info function and pass URL as an argument
    else
        echo "The flag is invalid. Please use the -z flag to add download .pdf files to a zip archive."    # Print error message
        exit 1  # Exit the script immediately
    fi
}

# Call the main function with command-line arguments
main "$@"   

exit 0

