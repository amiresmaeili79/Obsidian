Converting iPhone images to simple JPEG.

```bash
sudo apt install imagemagick libheif-examples
```

```bash
#!/bin/bash

# Define the output directory
OUTPUT_DIR="converted_jpegs"

# Create the directory if it doesn't exist
mkdir -p "$OUTPUT_DIR"

# Count for progress tracking
count=0

# Loop through all HEIC files (case-insensitive)
shopt -s nocaseglob
for file in *.heic; do
    # Check if any HEIC files actually exist to avoid errors
    [ -e "$file" ] || continue

    # Define the output filename (swap .heic for .jpg)
    filename=$(basename "$file")
    extension="${filename##*.}"
    raw_name="${filename%.*}"
    
    echo "Converting: $file..."
    
    # Use ImageMagick to convert
    # Note: Using 'magick' for v7+, use 'convert' if on v6
    magick "$file" "$OUTPUT_DIR/${raw_name}.jpg"
    
    ((count++))
done

echo "-------------------------------------"
echo "Done! $count files converted to /$OUTPUT_DIR"
```
