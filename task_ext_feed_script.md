# Task - External Data Feed Script

This task is to learn about setting up scripts and automating resource archiving and transfer across servers.

## 1. Psudo Code

The generalized high level language for the entire script

```

Initilize variables

for server in servers
  check if tempdir empty
  check if zip exists in srcDir
  copy zip and txt files from src to temp
  create dirDate in archive and backup temp there
  for fServer in sftpServers
    grab zip destinations from INSTRUCTION_* files (Assume each instruction has one zip destination)
    copy zip to server:destination
    keep track of successfull and failed zips and their instructions
    remove successfull zips and their instructions
    clean tempDir

```

## 2. Code Snippets

### 2.1 Variable initialization

Setting up constants and tracker variables

```bash
mailList="mokaddimul.kabir@therapservices.net"
fromAddr="mokaddimul.kabir@therapservices.net"
dir_date=$(date +%Y%m%d%H%M%S)
sourceDir=/usr/local/therap/external-data-feed
logFile=/interface/ext_data_feed/logs/ext_data_feed.log
archiveDir=/archive/ext_data_feed/backup
tempDir=/interface/ext_data_feed/temp
today=$(date | tr " " "-")
declare -i num_of_total_file=0
declare -i num_of_copied_file=0
declare -i tracker=0
```

### 2.2 Directory checks

Check temp if temp directory is empty

```bash
  hasDirsInTemp=$(ls -A "$tempDir" | wc -l)
  if [[ "$hasDirsInTemp" != "0" ]]  ##if 2
  then
    echo -e "\n"$tempDir" was supposed to be empty. But it is not." 1>> "$logFile" 2>&1
    tail -10000 "$logFile" | tr "\r" "\n" | tac | sed "/"$today"/q" |tac | mail -r "$fromAddr" -s "File copy Failed(ext_data_feed.sh)" $mailList
    exit 1
  fi
```

Check if source directory has zip files

```bash
  hasDirs=$(ssh batch@$batch_server "ls -A "$sourceDir"/*.zip | wc -l")
  if [[ "$hasDirs" == "0" ]]  ## if 1
  then
    echo -e "\nFound no files in $batch_server "$sourceDir" directory" 1>> "$logFile" 2>&1
    tracker=$(($tracker + 1)) ## Will add one for every empty server
    continue
  else
    echo -e "\nFound new files in $batch_server. List:" 1>> "$logFile" 2>&1
    ssh batch@"${batch_server}" "ls -lt "$sourceDir"/"*."{zip,txt}"  1>> "$logFile" 2>&1
  fi
```

### 2.3 Pull And Prep

Copy files from each server to the the interface server

```bash
  rsync -av  batch@"$batch_server":"$sourceDir"/"*."{zip,txt} "$tempDir"/   1>> "$logFile" 2>&1
  if [[ "$?" != "0" ]]
  then
    echo -e "\nCouldnt rsync from $batch_server.Exiting from the script" 1>> "$logFile" 2>&1
    tail -10000 "$logFile" | tr "\r" "\n" | tac | sed "/"$today"/q" |tac | mail -r "$fromAddr" -s "File copy Failed(ext_data_feed.sh)" $mailList
    exit 1
  else 
    echo -e "\nrsync was successful." 1>> "$logFile" 2>&1
  fi
```

Backup files into the archive directory

```bash
    mkdir "$archiveDir"/"$dir_date"
    echo -e "\nTaking backup of all the files to "$archiveDir"/"$dir_date" :" 1>> "$logFile" 2>&1
    cp -rp "$tempDir"/*  "${archiveDir}"/"${dir_date}"/
    if [[ "$?"  == "0" ]]
    then
      echo -e "\nCopied to "$archiveDir"" 1>> "$logFile" 2>&1
    else
      echo -e "\nCopying to "$archiveDir" Failed.Exiting.." 1>> "$logFile" 2>&1
      tail -10000 "$logFile" | tr "\r" "\n" | tac | sed "/"$today"/q" |tac | mail -r "$fromAddr" -s "File copy Failed(ext_data_feed.sh)" $mailList
      exit 1      
    fi
```

### 2.4 Push and Clean

First we grab the destination folder of each zip from instruction files. The files have content formatted as `ZIP_NAME=DESTINATION_FOLDER`

```bash
remoteDest=$(grep -h "$zipname" $tempDir/INSTRUCTION_*.txt | cut -d"=" -f2 ) #find out destination location from instruction files for every zip file. 
```

Now push the files to the `sftp` server to the appropriate destination. Along side, keep track of successful and failed files

```bash
   echo -e "\nTransferring all the zip files to sftp server" 1>> "$logFile" 2>&1
   for server in sftp01-ta
   do
     echo -e "\nCopying all the zip directories to "$server" " 1>> "$logFile" 2>&1
     for zipname in "${all_zip_files[@]}"  # adding all the zip files in an for loop
     do
       remoteDest=$(grep -h "$zipname" $tempDir/INSTRUCTION_*.txt | cut -d"=" -f2 ) #find out destination location from instruction files for every zip file. 
       num_of_total_file=$(($num_of_total_file + 1)) #For every zip file,we will increase the number_of_total_file count variable 
       if [[ -z "$remoteDest" ]]
       then
         echo -e "\nDestination directory name not found for $zipname file, Skipping this file" 1>> "$logFile" 2>&1
         all_failed_files+=("$zipname") # if dest dir not found for any zip file, then it will be added in failed array list
       else     
         echo -e "\n"${zipname}" file will go to "$server":"$remoteDest"" 1>> "$logFile" 2>&1
         #after getting destination info from INSTRUCTION file , we will copy zip file to the appropriate dir of sftp01-ta server
         rsync -avz "$tempDir"/"${zipname}" batch@"$server":"$remoteDest"/   1>> "$logFile" 2>&1
         if [[ "$?" == "0" ]]
         then
           echo -e "\nMoved "${zipname}" file in "$server" successfully" 1>> "$logFile" 2>&1
           all_copied_files+=("$zipname") #after successful copy, we will add this in all_copy_fileis array list
           num_of_copied_file=$(($num_of_copied_file + 1))   #Increase copied_file counter
           #For every successful copy, we also make an array list of INSTRUCTION files against copied zip file
           #successful_instruction_file+=("$(grep -l $zipname $tempDir/INSTRUCTION_*.txt )")
           successful_instruction_files+=("$(find $tempDir -name "INSTRUCTION_*.txt" -exec grep -ql $zipname {} \; -printf "%f\n" )")
         else
           echo -e "\nFailed to copy "${zipname}" file in the "$server":"$remoteDest".Skipping.." 1>> "$logFile" 2>&1
           all_failed_files+=("$zipname") #If any zip file is failed  to copy then it will be added in  failed array list
         fi
       fi       
     done
   done
```

Now we remove the ones that failed to be trainsfered

```bash
      delete_candidate_files=("${all_copied_files[@]}" "${successful_instruction_files[@]}") ## This array of files will be used to delete files from the source


      echo -e "\nDeleting copied directories and instruction.txt files from $batch_server server" 1>> "$logFile" 2>&1
      echo -e "\nList of files that will be deleted:\n" 1>> "$logFile" 2>&1
      #Print the list of deleting file in email
      for file in "${delete_candidate_files[@]}"
      do
        echo -e "$file"   1>> "$logFile" 2>&1
      done
      # Delete the listed file from source dir of batch server
      ssh batch@$batch_server  "cd $sourceDir; rm -v ${delete_candidate_files[@]} "
      if [[ "$?" != "0" ]]
      then
        echo -e "\nFailed to remove $sourceDir/$file from $batch_server server .Exiting.." 1>> "$logFile" 2>&1
        tail -10000 "$logFile" | tr "\r" "\n" | tac | sed "/"$today"/q" |tac | mail -r "$fromAddr" -s "File copy Failed(ext_data_feed.sh) $copied_file/$total_file " $mailList
        exit 1
      else 
        echo -e "\nAll listed files successfully Deleted from source dir of $batch_server server."   1>> "$logFile" 2>&1
      fi
```

Now we clean up temp directory as its backed up in archive folder

```bash
    echo -e "\nClearing the "$tempDir" directory " 1>> "$logFile" 2>&1
    rm -r "$tempDir"/*   1>> "$logFile" 2>&1
    if [[ "$?" != "0" ]]
    then
      echo -e "\nFailed to remove all the directories in the "$tempDir" .Exiting.." 1>> "$logFile" 2>&1
      tail -10000 "$logFile" | tr "\r" "\n" | tac | sed "/"$today"/q" |tac | mail -r "$fromAddr" -s "File copy Failed(ext_data_feed.sh) $copied_file/$total_file" $mailList
      exit 1
    else 
      echo -e "\nCleanup Done"   1>> "$logFile" 2>&1
    fi
```

## 3 Modified Code Snippets

Modified to allow rsync to the destination

```bash
         echo -e "\n"${zipname}" file will go to "$server":"$remoteDest"" 1>> "$logFile" 2>&1
         #after getting destination info from INSTRUCTION file , we will copy zip file to the appropriate dir of sftp01-ta server
         rsync -avz "$tempDir"/"${zipname}" batch@"$server":"$remoteDest"/   1>> "$logFile" 2>&1
```

Modified to print out total number of file copies

```bash
echo -e "\nList of files that successfully copied to the sftp server:\n" 1>> "$logFile" 2>&1
c_num=0
for c_file in "${all_copied_files[@]}"
do
  echo -e "$c_file"   1>> "$logFile" 2>&1
  c_num=$(( c_cnum + 1))
done
echo -e "\nTotal number of files copied: $c_num\n" 1>> "$logFile" 2>&1

```

## 4. Population Script

This script was created to generate sample data files in src directory

```bash
#!/bin/bash

# Declare needed variables
POPULATION_DIR="/usr/local/therap/external-data-feed"
FILE_NUM=$(( $RANDOM % 11 + 20 ))
OUTFOLDERS=("DemographicProfileExport"  "MedProfileExport"  "MetaProfileExport")

# CLean up directory
rm -rf "$POPULATION_DIR/*"

for ((i=1; i<=FILE_NUM; i++))
do
  # Randomly create file or folder
  if [[ "$(( $RANDOM%3 ))" != "1" ]];then
	 # Set random file data name and zip name
	 FILE_NAME="/tmp/$RANDOM$RANDOM$RANDOM.data"
	 ZIP_NAME="$RANDOM$RANDOM$RANDOM.zip"

	 # Create empty data file, zip it and create instruction file for it
	 touch "$FILE_NAME"
	 zip "$POPULATION_DIR/$ZIP_NAME" "$FILE_NAME"
	 rm "$FILE_NAME"
	 echo "$POPULATION_DIR/$ZIP_NAME=/sftp/ddncs-home/ddncs/${OUTFOLDERS[$(($RANDOM % 3))]}" > "$POPULATION_DIR/INSTRUCTION_$RANDOM$RANDOM$RANDOM.txt"
  else
	 mkdir -p "$POPULATION_DIR/$RANDOM$RANDOM$RANDOM"
  fi
done
```

## 5. Output

The log generated by the script on a single run

```bash
Wed-Apr--1-01:52:25-+06-2026 
Hostname:
kabir8

Script name:ext_data_feed.sh
Purpose:This script will rsync/copy recipient-name_timestamp.zip and INSTRUCTION_*.txt  files from xxbatch11:/usr/local/therap/external-data-feed directory.Then put the zip files in  sftp server directories specified in INSTRUCTION*.txt file

Found new files in tabatch11. List:
-rw-r--r--. 1 root root 198 Apr  1 01:52 /usr/local/therap/external-data-feed/549351522668.zip
-rw-r--r--. 1 root root 102 Apr  1 01:52 /usr/local/therap/external-data-feed/INSTRUCTION_154201983329506.txt
-rw-r--r--. 1 root root  97 Apr  1 01:52 /usr/local/therap/external-data-feed/INSTRUCTION_25743560020394.txt
-rw-r--r--. 1 root root 198 Apr  1 01:52 /usr/local/therap/external-data-feed/254552229931622.zip
-rw-r--r--. 1 root root 192 Apr  1 01:52 /usr/local/therap/external-data-feed/1937965584855.zip
-rw-r--r--. 1 root root  95 Apr  1 01:52 /usr/local/therap/external-data-feed/INSTRUCTION_2255329180838.txt
-rw-r--r--. 1 root root 198 Apr  1 01:52 /usr/local/therap/external-data-feed/28435376632501.zip
-rw-r--r--. 1 root root  96 Apr  1 01:52 /usr/local/therap/external-data-feed/INSTRUCTION_113012357620799.txt
-rw-r--r--. 1 root root 103 Apr  1 01:52 /usr/local/therap/external-data-feed/INSTRUCTION_44202480513817.txt
-rw-r--r--. 1 root root 194 Apr  1 01:52 /usr/local/therap/external-data-feed/4354252775031.zip
-rw-r--r--. 1 root root  97 Apr  1 01:52 /usr/local/therap/external-data-feed/INSTRUCTION_206752710611296.txt
-rw-r--r--. 1 root root 198 Apr  1 01:52 /usr/local/therap/external-data-feed/32078136841878.zip
-rw-r--r--. 1 root root  97 Apr  1 01:52 /usr/local/therap/external-data-feed/INSTRUCTION_323603141316005.txt
-rw-r--r--. 1 root root 196 Apr  1 01:52 /usr/local/therap/external-data-feed/183682617619713.zip
-rw-r--r--. 1 root root 196 Apr  1 01:52 /usr/local/therap/external-data-feed/66222670319763.zip
-rw-r--r--. 1 root root  97 Apr  1 01:52 /usr/local/therap/external-data-feed/INSTRUCTION_238842767827495.txt
-rw-r--r--. 1 root root 103 Apr  1 01:52 /usr/local/therap/external-data-feed/INSTRUCTION_1849076813510.txt
-rw-r--r--. 1 root root 198 Apr  1 01:52 /usr/local/therap/external-data-feed/3034671478836.zip
-rw-r--r--. 1 root root 105 Apr  1 01:52 /usr/local/therap/external-data-feed/INSTRUCTION_2252319164612.txt
-rw-r--r--. 1 root root 198 Apr  1 01:52 /usr/local/therap/external-data-feed/306481191816707.zip
-rw-r--r--. 1 root root  96 Apr  1 01:52 /usr/local/therap/external-data-feed/INSTRUCTION_22772186674955.txt
-rw-r--r--. 1 root root 198 Apr  1 01:52 /usr/local/therap/external-data-feed/9391104118566.zip
-rw-r--r--. 1 root root 196 Apr  1 01:52 /usr/local/therap/external-data-feed/107681935812220.zip
-rw-r--r--. 1 root root  97 Apr  1 01:52 /usr/local/therap/external-data-feed/INSTRUCTION_12351559827601.txt
-rw-r--r--. 1 root root  95 Apr  1 01:52 /usr/local/therap/external-data-feed/INSTRUCTION_244531622115866.txt
-rw-r--r--. 1 root root 198 Apr  1 01:52 /usr/local/therap/external-data-feed/4953297674688.zip
-rw-r--r--. 1 root root 198 Apr  1 01:52 /usr/local/therap/external-data-feed/300261041316120.zip
-rw-r--r--. 1 root root  98 Apr  1 01:52 /usr/local/therap/external-data-feed/INSTRUCTION_4847144333893.txt
-rw-r--r--. 1 root root  98 Apr  1 01:52 /usr/local/therap/external-data-feed/INSTRUCTION_115969365610.txt
-rw-r--r--. 1 root root 196 Apr  1 01:52 /usr/local/therap/external-data-feed/258592933610309.zip

Copying all .zip and .txt files  from /usr/local/therap/external-data-feed of tabatch11
receiving incremental file list
107681935812220.zip
183682617619713.zip
1937965584855.zip
254552229931622.zip
258592933610309.zip
28435376632501.zip
300261041316120.zip
3034671478836.zip
306481191816707.zip
32078136841878.zip
4354252775031.zip
4953297674688.zip
549351522668.zip
66222670319763.zip
9391104118566.zip
INSTRUCTION_113012357620799.txt
INSTRUCTION_115969365610.txt
INSTRUCTION_12351559827601.txt
INSTRUCTION_154201983329506.txt
INSTRUCTION_1849076813510.txt
INSTRUCTION_206752710611296.txt
INSTRUCTION_2252319164612.txt
INSTRUCTION_2255329180838.txt
INSTRUCTION_22772186674955.txt
INSTRUCTION_238842767827495.txt
INSTRUCTION_244531622115866.txt
INSTRUCTION_25743560020394.txt
INSTRUCTION_323603141316005.txt
INSTRUCTION_44202480513817.txt
INSTRUCTION_4847144333893.txt

sent 594 bytes  received 6,627 bytes  14,442.00 bytes/sec
total size is 4,428  speedup is 0.61

rsync was successful.

List of all the zip file contents

Zipped file Name: 107681935812220.zip

  Length      Date    Time    Name
---------  ---------- -----   ----
        0  04-01-2026 01:52   tmp/25412120765677.data
---------                     -------
        0                     1 file

Zipped file Name: 183682617619713.zip

  Length      Date    Time    Name
---------  ---------- -----   ----
        0  04-01-2026 01:52   tmp/45873255417294.data
---------                     -------
        0                     1 file

Zipped file Name: 1937965584855.zip

  Length      Date    Time    Name
---------  ---------- -----   ----
        0  04-01-2026 01:52   tmp/852824586597.data
---------                     -------
        0                     1 file

Zipped file Name: 254552229931622.zip

  Length      Date    Time    Name
---------  ---------- -----   ----
        0  04-01-2026 01:52   tmp/316472196224276.data
---------                     -------
        0                     1 file

Zipped file Name: 258592933610309.zip

  Length      Date    Time    Name
---------  ---------- -----   ----
        0  04-01-2026 01:52   tmp/31886239523921.data
---------                     -------
        0                     1 file

Zipped file Name: 28435376632501.zip

  Length      Date    Time    Name
---------  ---------- -----   ----
        0  04-01-2026 01:52   tmp/160581719129698.data
---------                     -------
        0                     1 file

Zipped file Name: 300261041316120.zip

  Length      Date    Time    Name
---------  ---------- -----   ----
        0  04-01-2026 01:52   tmp/132932324223192.data
---------                     -------
        0                     1 file

Zipped file Name: 3034671478836.zip

  Length      Date    Time    Name
---------  ---------- -----   ----
        0  04-01-2026 01:52   tmp/123422915032412.data
---------                     -------
        0                     1 file

Zipped file Name: 306481191816707.zip

  Length      Date    Time    Name
---------  ---------- -----   ----
        0  04-01-2026 01:52   tmp/163683225313307.data
---------                     -------
        0                     1 file

Zipped file Name: 32078136841878.zip

  Length      Date    Time    Name
---------  ---------- -----   ----
        0  04-01-2026 01:52   tmp/123451656330755.data
---------                     -------
        0                     1 file

Zipped file Name: 4354252775031.zip

  Length      Date    Time    Name
---------  ---------- -----   ----
        0  04-01-2026 01:52   tmp/2521187363279.data
---------                     -------
        0                     1 file

Zipped file Name: 4953297674688.zip

  Length      Date    Time    Name
---------  ---------- -----   ----
        0  04-01-2026 01:52   tmp/318671522112869.data
---------                     -------
        0                     1 file

Zipped file Name: 549351522668.zip

  Length      Date    Time    Name
---------  ---------- -----   ----
        0  04-01-2026 01:52   tmp/203343222126345.data
---------                     -------
        0                     1 file

Zipped file Name: 66222670319763.zip

  Length      Date    Time    Name
---------  ---------- -----   ----
        0  04-01-2026 01:52   tmp/10502886427521.data
---------                     -------
        0                     1 file

Zipped file Name: 9391104118566.zip

  Length      Date    Time    Name
---------  ---------- -----   ----
        0  04-01-2026 01:52   tmp/223632837711818.data
---------                     -------
        0                     1 file

Taking backup of all the files to /archive/ext_data_feed/backup/20260401015225 :

Copied to /archive/ext_data_feed/backup

Transferring all the zip files to sftp server

Copying all the zip directories to sftp01-ta 

107681935812220.zip file will go to sftp01-ta:/sftp/ddncs-home/ddncs/MedProfileExport
sending incremental file list
107681935812220.zip

sent 314 bytes  received 35 bytes  698.00 bytes/sec
total size is 196  speedup is 0.56

Moved 107681935812220.zip file in sftp01-ta successfully

183682617619713.zip file will go to sftp01-ta:/sftp/ddncs-home/ddncs/MedProfileExport
sending incremental file list
183682617619713.zip

sent 314 bytes  received 35 bytes  698.00 bytes/sec
total size is 196  speedup is 0.56

Moved 183682617619713.zip file in sftp01-ta successfully

1937965584855.zip file will go to sftp01-ta:/sftp/ddncs-home/ddncs/MedProfileExport
sending incremental file list
1937965584855.zip

sent 308 bytes  received 35 bytes  686.00 bytes/sec
total size is 192  speedup is 0.56

Moved 1937965584855.zip file in sftp01-ta successfully

254552229931622.zip file will go to sftp01-ta:/sftp/ddncs-home/ddncs/MedProfileExport
sending incremental file list
254552229931622.zip

sent 316 bytes  received 35 bytes  234.00 bytes/sec
total size is 198  speedup is 0.56

Moved 254552229931622.zip file in sftp01-ta successfully

258592933610309.zip file will go to sftp01-ta:/sftp/ddncs-home/ddncs/MetaProfileExport
sending incremental file list
258592933610309.zip

sent 314 bytes  received 35 bytes  698.00 bytes/sec
total size is 196  speedup is 0.56

Moved 258592933610309.zip file in sftp01-ta successfully

28435376632501.zip file will go to sftp01-ta:/sftp/ddncs-home/ddncs/MedProfileExport
sending incremental file list
28435376632501.zip

sent 315 bytes  received 35 bytes  700.00 bytes/sec
total size is 198  speedup is 0.57

Moved 28435376632501.zip file in sftp01-ta successfully

300261041316120.zip file will go to sftp01-ta:/sftp/ddncs-home/ddncs/MetaProfileExport
sending incremental file list
300261041316120.zip

sent 316 bytes  received 35 bytes  702.00 bytes/sec
total size is 198  speedup is 0.56

Moved 300261041316120.zip file in sftp01-ta successfully

3034671478836.zip file will go to sftp01-ta:/sftp/ddncs-home/ddncs/DemographicProfileExport
sending incremental file list
3034671478836.zip

sent 314 bytes  received 35 bytes  698.00 bytes/sec
total size is 198  speedup is 0.57

Moved 3034671478836.zip file in sftp01-ta successfully

306481191816707.zip file will go to sftp01-ta:/sftp/ddncs-home/ddncs/DemographicProfileExport
sending incremental file list
306481191816707.zip

sent 316 bytes  received 35 bytes  702.00 bytes/sec
total size is 198  speedup is 0.56

Moved 306481191816707.zip file in sftp01-ta successfully

32078136841878.zip file will go to sftp01-ta:/sftp/ddncs-home/ddncs/MetaProfileExport
sending incremental file list
32078136841878.zip

sent 315 bytes  received 35 bytes  700.00 bytes/sec
total size is 198  speedup is 0.57

Moved 32078136841878.zip file in sftp01-ta successfully

4354252775031.zip file will go to sftp01-ta:/sftp/ddncs-home/ddncs/DemographicProfileExport
sending incremental file list
4354252775031.zip

sent 310 bytes  received 35 bytes  230.00 bytes/sec
total size is 194  speedup is 0.56

Moved 4354252775031.zip file in sftp01-ta successfully

4953297674688.zip file will go to sftp01-ta:/sftp/ddncs-home/ddncs/MedProfileExport
sending incremental file list
4953297674688.zip

sent 314 bytes  received 35 bytes  698.00 bytes/sec
total size is 198  speedup is 0.57

Moved 4953297674688.zip file in sftp01-ta successfully

549351522668.zip file will go to sftp01-ta:/sftp/ddncs-home/ddncs/DemographicProfileExport
sending incremental file list
549351522668.zip

sent 313 bytes  received 35 bytes  696.00 bytes/sec
total size is 198  speedup is 0.57

Moved 549351522668.zip file in sftp01-ta successfully

66222670319763.zip file will go to sftp01-ta:/sftp/ddncs-home/ddncs/MetaProfileExport
sending incremental file list
66222670319763.zip

sent 313 bytes  received 35 bytes  696.00 bytes/sec
total size is 196  speedup is 0.56

Moved 66222670319763.zip file in sftp01-ta successfully

9391104118566.zip file will go to sftp01-ta:/sftp/ddncs-home/ddncs/MetaProfileExport
sending incremental file list
9391104118566.zip

sent 314 bytes  received 35 bytes  698.00 bytes/sec
total size is 198  speedup is 0.57

Moved 9391104118566.zip file in sftp01-ta successfully

Deleting copied directories and instruction.txt files from tabatch11 server

List of files that will be deleted:

107681935812220.zip
183682617619713.zip
1937965584855.zip
254552229931622.zip
258592933610309.zip
28435376632501.zip
300261041316120.zip
3034671478836.zip
306481191816707.zip
32078136841878.zip
4354252775031.zip
4953297674688.zip
549351522668.zip
66222670319763.zip
9391104118566.zip
INSTRUCTION_12351559827601.txt
INSTRUCTION_323603141316005.txt
INSTRUCTION_2255329180838.txt
INSTRUCTION_25743560020394.txt
INSTRUCTION_115969365610.txt
INSTRUCTION_113012357620799.txt
INSTRUCTION_4847144333893.txt
INSTRUCTION_1849076813510.txt
INSTRUCTION_2252319164612.txt
INSTRUCTION_206752710611296.txt
INSTRUCTION_44202480513817.txt
INSTRUCTION_244531622115866.txt
INSTRUCTION_154201983329506.txt
INSTRUCTION_238842767827495.txt
INSTRUCTION_22772186674955.txt

All listed files successfully Deleted from source dir of tabatch11 server.

Clearing the /interface/ext_data_feed/temp directory 

Cleanup Done

Found no files in tbbatch11 /usr/local/therap/external-data-feed directory

List of files that successfully copied to the sftp server:

107681935812220.zip
183682617619713.zip
1937965584855.zip
254552229931622.zip
258592933610309.zip
28435376632501.zip
300261041316120.zip
3034671478836.zip
306481191816707.zip
32078136841878.zip
4354252775031.zip
4953297674688.zip
549351522668.zip
66222670319763.zip
9391104118566.zip

Total number of files copied: 15


List of files that failed to be copied in the sftp server:


Everything went well
```
