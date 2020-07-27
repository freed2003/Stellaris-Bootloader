# design-challenge-error-404-brain-not-found

## bl_build: 
  - Creates a seed for the key stream cipher
  - Stores this seed with the rest of the code for the bootloader, as well as in the secret_build_output.txt file. 
  - The bootloader generates a main.bin file which has contents of flash memory

## fw_protect:
  - Reads in infile, outfile, version number, and release message
    -```python fw_protect --infile --outfile --version --message```
  ### Variables:
   - infile (Specifies file to protect)
   - outfile (Specifies file to write to)
   - message (Uses as release message)
   - version (Specifies firmware version number)
    
  ### Protocol:
   1. Combines release message and firmware while adding a nullbyte as a terminator
   2. Generates key from stream cipher
   3. Creates METADATA(version, size, and HMACS)
   4. Creates an HMAC of the firmware and one of the Version and Size using two different keys and adds to METADATA
   5. Breaks the firmware into frames with the first 16 bytes being IV, next 2 being size, at most 1kb of cipher text, and the tag as the last 16
   6. Writes METADATA and framed firmware with release message to outdata

## fw_update:
  - Sends the data from the infile to the bootloader
    -```python fw_update --port, --firmware```
  ### Variables:
   - port (Port to write to)
   - firmware (Chooses file to install)
   - [debug] (Turns on debug mode)
   
  ### Protocol:
   1. Handshake (update tool sends U, bootloader sends a U back)
   2. Update tool sends over the metadata package (in frames if necessary)
   4. Waits for "OK" byte before sending next frame
   5. Send a black frame to indicate it is done sending frames
   
   **_NOTE:_**    If debug is turned on extra data is printed to screen

## bootloader.c:
  - Loads or boots firmware
  - Rejects older versions or invalid firmware
  - Generates key with stream cipher
  - Prints to UART2
  ### Protocol:
   - Waits for byte to specify mode: "U" for update; "B" for boot
   - 0x20 to UART0 resets device
   #### Update mode:
   1. Reads METADATA from update tool
   2. Creates key with stream cipher
   3. Verifies METADATA HMAC
   4. If correct, writes METADATA to address 0xFC00
   5. Reads each frame of firmware
   6. Uses provided IV to decrypt data
   7. Flashes to storage in pages starting at address 0x1000
   8. Verifies firmware HMAC
   9. If wrong, erases firmware
   1. Sends "OK" to indicate end of update
   #### Boot mode:
   1. Prints release message
   2. Boots firmware
   
## Using the bootloader
   1. Navigate to /firmware/firmware and run ```make```
   2. Navigate to /tools and run ```python bl_build``` to compile the bootloader
   3. Run ```python bl_emulate``` to run the bootloader
   4. Open a UART with ```miniterm /embsec/UARTX where X is the UART number
## Using the firmware tools
   1. Navigate to /tools and run the fw_protect tool to protect the firmware
   2. Run the fw_update tool to attempt to update the booloader
   
   **_NOTE:_**    bl_emulate must be running before attempting to update the bootloader
