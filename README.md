Client Onboarding Documentation For Rhapsody: Step-by-Step Processing & Automation Guide

This guide provides a clear, step-by-step walkthrough for onboarding new clients onto our automated file ingestion system. Follow the steps sequentially to configure, test, and deploy daily processing scripts.

 

1. System Flow Overview 

The diagram below shows how insurance claim files travel from the client into our systems every day:

STAGE 1: Ingress

 

STAGE 2: Scan & Log

 

STAGE 3: Delivery

 

Client Drop-off

Files arrive on the SFTP share or network folder.

➔

Script Execution

Script reads files, counts 835/837 records, and updates the daily CSV report.

➔

Rhapsody Core

Files are renamed with a prefix and moved to the central input folder.



 

2. Selecting the Processing Method

Before creating a script, identify the file delivery style the client requires. Choose one of the three active methods below:

Method

Operational Steps

Active Reference

 

Method A: Direct Move

1. Read network folder directly

2. Archive file copies

3. Move files straight to Rhapsody

PHMG (Palomar)

Method B: Zip Extraction

1. Python utility extracts compressed .zip folders

2. Delete original zip containers

3. Batch script processes the loose extracts

PDWD (Platinum Derm)

Method C: Local Staging

1. Script 1 copies files to a local server folder

2. Script 2 counts records and routes to Rhapsody

3. Staging folder is wiped completely clean

SKY (Skylakes)



 

3. Server & Client Folder Inventory (Server 10.2.1.83)

All file transfer automation tasks are hosted on production server 10.2.1.83. Below is the complete directory inventory of active client folders mapped in Windows Task Scheduler. Their corresponding processing scripts reside under C:\AutoMoveBatchFiles\. 

Also the script details can be found in task scheduler by clicking on respective client folder and and then clicking on job to know the script related details as shown below in Actions tab:



Folders

Task Scheduler Folder / Client ID

Operational Notes & Script Target Location

 









AXIA

Legacy / Active client automation stream script located in C:\AutoMoveBatchFiles\

CMM

Active client automation stream script located in C:\AutoMoveBatchFiles\

GALEN

Active client automation stream script located in C:\AutoMoveBatchFiles\

GIA

Active client automation stream script located in C:\AutoMoveBatchFiles\. 

Uses Method C (Two-Script Fetching & Local Staging wipe).

PDWD

Platinum Derm - Uses Method B (Python Zip Extraction + Processing script).

PHMG

Palomar Health - Uses Method A (Standalone Single-Pass Network Move script).

RVHT

Active client automation stream script located in C:\AutoMoveBatchFiles\

Uses Method C (Two-Script Fetching & Local Staging wipe).

SKY

Skylakes - Uses Method C (Two-Script Fetching & Local Staging wipe).

THC

Active client automation stream script located in C:\AutoMoveBatchFiles\

Uses Method C (Two-Script Fetching & Local Staging wipe).

WWMG

Active client automation stream script located in C:\AutoMoveBatchFiles\

Uses Method C (Two-Script Fetching & Local Staging wipe).







 

4. Step-by-Step Setup Procedure

Follow these exact sequential phases to onboard the new client successfully.

Step 1: Check the Content Separator

Open a sample data file from the client using Notepad. Look at how the loops are split:

Asterisk Separation: If lines contain ST*835, you must use ST\*835 inside your script search commands. The backslash tells Windows to search for a literal asterisk instead of treating it as a wildcard.

Pipe Separation: If lines contain ST|835, change the script search commands to use ST|835.

Step 2: Create the Script File

Copy the code block from Section 4 that corresponds to your chosen processing method. Open a new Notepad file, paste the template, and edit the paths at the very top. Always save your finalized production scripts inside the standard server directory: C:\AutoMoveBatchFiles\.

Step 3: Perform a Manual Test Run

Place 2 or 3 test files inside the client's inbound folder. Navigate to C:\AutoMoveBatchFiles\ and double-click the script to run it manually. Confirm two outcomes:

The files leave the source folder and arrive in C:\Rhapsodyprod\AllClients_835_837\Input\ with the correct shorthand client prefix attached to the front of the filename.

A daily CSV spreadsheet log updates inside the designated logging path, capturing accurate file counts and sizes.

Step 4: Configure the Task Scheduler

Open the Windows Task Scheduler console on the server and construct a new daily task:

Security Settings: Select "Run whether user is logged on or not" and check "Run with highest privileges". Assign the task to an administrative service profile that has permissions to modify local disks and network UNC paths.

Action Settings: Choose "Start a program" and browse to your script. You must paste C:\AutoMoveBatchFiles directly into the "Start in (optional)" field. Leaving this blank will cause the script to fail when executing in the background.

 



5. Code Templates

Template A: Direct Move Script (PHMG Style)

@echo off

setlocal enabledelayedexpansion



:: ============================================================

:: CONFIGURATION

:: ============================================================

set "source_folder=\\10.2.1.27\YourClient\Inbound"

set "destination_folder=C:\Rhapsodyprod\AllClients_835_837\Input"

set "sftp_archive_folder=\\10.2.1.27\YourClient\Archive"

set "log_path=C:\Rhapsodyprod\Log_files\yourclient"

set "client_id=NEWCLIENT"

set "prefix=NEWCLIENT_"



:: ============================================================

:: AUTOMATIC LOGIC

:: ============================================================

for /f "tokens=2 delims==" %%I in ('"wmic os get localdatetime /value"') do set datetime=%%I

set "date=%datetime:~4,2%-%datetime:~6,2%-%datetime:~0,4%"

set "log_file=%log_path%\%date%_report_log_%client_id%.csv"

set "time_now=%time:~0,8%"



if not exist "%destination_folder%" mkdir "%destination_folder%"

if not exist "%sftp_archive_folder%\%date%" mkdir "%sftp_archive_folder%\%date%"

if not exist "%log_path%" mkdir "%log_path%"



set "archive_path=%sftp_archive_folder%\%date%"

set received_count=0

set pushed_count=0

set total_size_kb_rec=0

set total_size_kb_dest=0

set st835_count=0

set st837_count=0



if not exist "%log_file%" (

    echo FileName,Status,Size ^(KB^),TimeStamp,SourcePath,DestinationPath,835_Count,837_Count > "%log_file%"

)



for /r "%source_folder%" %%f in (*.*) do (

    set "source_file_path=%%f"

    set "filename=%%~nxf"

    set "newname=%prefix%!filename!"

    set "dest_file_path=%destination_folder%\!newname!"



    for %%A in ("!source_file_path!") do set /a size_kb_rec=%%~zA / 1024

    set "is_835=0"

    set "is_837=0"



    findstr /i "ST\*835" "%%f" >nul

    if !errorlevel! equ 0 ( set /a st835_count+=1 & set "is_835=1" )

    findstr /i "ST\*837" "%%f" >nul

    if !errorlevel! equ 0 ( set /a st837_count+=1 & set "is_837=1" )



    copy /Y "%%f" "%archive_path%" >nul

    echo !filename!,Received,!size_kb_rec!,%date% %time_now%,"%%f","%archive_path%\!filename!",!is_835!,!is_837! >> "%log_file%"

    set /a received_count+=1

    set /a total_size_kb_rec+=!size_kb_rec!



    move /Y "%%f" "!dest_file_path!" >nul



    for %%B in ("!dest_file_path!") do set /a size_kb_dest=%%~zB / 1024

    echo !newname!,Pushed,!size_kb_dest!,%date% %time_now%,"%%f","!dest_file_path!",!is_835!,!is_837! >> "%log_file%"

    set /a pushed_count+=1

    set /a total_size_kb_dest+=!size_kb_dest!

)



echo Summary for %date% (Daily Report) >> "%log_file%"

echo Files Received,!received_count! >> "%log_file%"

echo Files Pushed,!pushed_count! >> "%log_file%"

echo Total Size Received (KB),!total_size_kb_rec! >> "%log_file%"

echo Total Size Pushed (KB),!total_size_kb_dest! >> "%log_file%"

echo 835 Count,!st835_count! >> "%log_file%"

echo 837 Count,!st837_count! >> "%log_file%"



echo Summary for GCP >> "%log_file%"

echo %client_id%,!received_count!,%date% %time_now%,SFTP,Rhapsody,!st835_count!,!st837_count! >> "%log_file%"

endlocal

Template B: Zip Extraction Scripts (PDWD Style)

Step 1: Python Unzipper Utility (Save as C:\AutoMoveBatchFiles\Unzip_YourClient.py)

import os

import zipfile



def unzip_and_delete(source_folder, output_folder):

    if not os.path.exists(output_folder):

        os.makedirs(output_folder)



    for filename in os.listdir(source_folder):

        if filename.endswith(".zip"):

            file_path = os.path.join(source_folder, filename)

            try:

                with zipfile.ZipFile(file_path, 'r') as zip_ref:

                    zip_ref.extractall(output_folder)

                os.remove(file_path)

            except Exception as e:

                print(f"Error processing {filename}: {e}")



if __name__ == "__main__":

    unzip_and_delete(source_folder=r"\\10.2.1.27\YourClient\Inbound\837", output_folder=r"\\10.2.1.27\YourClient\Inbound\837\Extracts")

    unzip_and_delete(source_folder=r"\\10.2.1.27\YourClient\Inbound\835", output_folder=r"\\10.2.1.27\YourClient\Inbound\835\Extracts")

Step 2: File Processing Script

Deploy Template A immediately after the Python unzipper task completes to move, check, and log the unzipped file contents.

Template C: Local Staging Scripts (SKY Style)

Script 1: Fetch and Stage Files (Save as C:\AutoMoveBatchFiles\Fetch_YourClient.bat)

@echo off

set "source_folder=\\10.2.1.27\YourClient\Inbound"

set "destination_folder=C:\YOURCLIENT_StagingFiles"

set "archive_folder=C:\AutoMoveBatchFiles\Clientwise_Archive\YOURCLIENT_Archive"

set "sftp_archive_folder=\\10.2.1.27\YourClient\Archive"



for /f "tokens=2 delims==" %%I in ('"wmic os get localdatetime /value"') do set datetime=%%I

set date=%datetime:~4,2%-%datetime:~6,2%-%datetime:~0,4%



if not exist "%archive_folder%\%date%" mkdir "%archive_folder%\%date%"

set "archive_path=%archive_folder%\%date%"



ROBOCOPY "%source_folder%" "\\10.2.1.27\YourClient\Inbound_temp" /E

ROBOCOPY "%source_folder%" "%archive_path%" /E

ROBOCOPY "%source_folder%" "%destination_folder%" /E /mov

ROBOCOPY "%archive_folder%" "%sftp_archive_folder%" /MOVE /E

Script 2: Local Processing and Cleanup (Save as C:\AutoMoveBatchFiles\Process_YourClient.bat)

@echo off

setlocal enabledelayedexpansion



set "source_folder=C:\YOURCLIENT_StagingFiles"

set "destination_folder=C:\Rhapsodyprod\AllClients_835_837\Input"

set "log_path=C:\Rhapsodyprod\Log_files\yourclient"

set "client_id=NEWCLIENT"

set "prefix=NEWCLIENT_"



for /f "tokens=2 delims==" %%I in ('"wmic OS Get localdatetime /value"') do set datetime=%%I

set "date_today=%datetime:~4,2%-%datetime:~6,2%-%datetime:~0,4%"

set "log_file=%log_path%\%date_today%_report_log_%client_id%.csv"

set "time_now=%time:~0,8%"



set received_count=0

set pushed_count=0

set total_size_kb_rec=0

set total_size_kb_dest=0

set st835_count=0

set st837_count=0



if not exist "%log_file%" (

    echo FileName,Status,Size ^(KB^),TimeStamp,SourcePath,DestinationPath,835_Count,837_Count > "%log_file%"

)

if not exist "%destination_folder%" mkdir "%destination_folder%"



for /r "%source_folder%" %%F in (*.*) do (

    set "source_file_path=%%F"

    set "filename=%%~nxF"

    set "new_name=%prefix%!filename!"

    set "destination_file_path=%destination_folder%\!new_name!"



    for %%A in ("!source_file_path!") do set /a size_kb_rec=%%~zA / 1024



    findstr /i "ST\*835" "!source_file_path!" >nul

    if !errorlevel! equ 0 ( set /a st835_count+=1 )

    findstr /i "ST\*837" "!source_file_path!" >nul

    if !errorlevel! equ 0 ( set /a st837_count+=1 )



    copy /y "!source_file_path!" "!destination_file_path!" > nul



    echo !filename!,Received,!size_kb_rec!,%date_today% %time_now%,!source_file_path!,!destination_file_path!,!st835_count!,!st837_count! >> "%log_file%"

    set /a received_count+=1

    set /a total_size_kb_rec+=!size_kb_rec!

)



for /r "%destination_folder%" %%F in (*.*) do (

    set "destination_file_path=%%F"

    set "filename=%%~nxF"

    for %%A in ("!destination_file_path!") do set /a size_kb_dest=%%~zA / 1024

    echo !filename!,Pushed,!size_kb_dest!,%date_today% %time_now%,%source_folder%,!destination_file_path!,!st835_count!,!st837_count! >> "%log_file%"

    set /a pushed_count+=1

    set /a total_size_kb_dest+=!size_kb_dest!

)



echo Summary for %date_today% (Daily Report) >> "%log_file%"

echo Files Received,!received_count! >> "%log_file%"

echo Files Pushed,!pushed_count! >> "%log_file%"

echo Total Size Received (KB),!total_size_kb_rec! >> "%log_file%"

echo Total Size Pushed (KB),!total_size_kb_dest! >> "%log_file%"

echo 835 Count,!st835_count! >> "%log_file%"

echo 837 Count,!st837_count! >> "%log_file%"



echo Summary for GCP >> "%log_file%"

echo %client_id%,!received_count!,%date_today% !time_now!,SFTP,Rhapsody,!st835_count!,!st837_count! >> "%log_file%"



del /Q "%source_folder%\*.*"

endlocal

 

5. Key Contacts



ROLE

NAME

EMAIL

NOTE

File Support

Ravish Khokle

ravish.khokle@ikshealth.com 

File Support

File Transfer Support

Jay Bhoyar

jay.bhoyar@ikshealth.com 

File Transfer Support 

Rhapsody Support

Vipin, Ashwini

vipin.kumar@ikshealth.com, ashwini.nadar@ikshealth.com 

Rhapsody Processing Support






Client Onboarding Documentation For Rhapsody: Step-by-Step Processing & Automation Guide

This guide provides a clear, step-by-step walkthrough for onboarding new clients onto our automated file ingestion system. Follow the steps sequentially to configure, test, and deploy daily processing scripts.

 

1. System Flow Overview 

The diagram below shows how insurance claim files travel from the client into our systems every day:

STAGE 1: Ingress

 

STAGE 2: Scan & Log

 

STAGE 3: Delivery

 

Client Drop-off

Files arrive on the SFTP share or network folder.

➔

Script Execution

Script reads files, counts 835/837 records, and updates the daily CSV report.

➔

Rhapsody Core

Files are renamed with a prefix and moved to the central input folder.



 

2. Selecting the Processing Method

Before creating a script, identify the file delivery style the client requires. Choose one of the three active methods below:

Method

Operational Steps

Active Reference

 

Method A: Direct Move

1. Read network folder directly

2. Archive file copies

3. Move files straight to Rhapsody

PHMG (Palomar)

Method B: Zip Extraction

1. Python utility extracts compressed .zip folders

2. Delete original zip containers

3. Batch script processes the loose extracts

PDWD (Platinum Derm)

Method C: Local Staging

1. Script 1 copies files to a local server folder

2. Script 2 counts records and routes to Rhapsody

3. Staging folder is wiped completely clean

SKY (Skylakes)



 

3. Server & Client Folder Inventory (Server 10.2.1.83)

All file transfer automation tasks are hosted on production server 10.2.1.83. Below is the complete directory inventory of active client folders mapped in Windows Task Scheduler. Their corresponding processing scripts reside under C:\AutoMoveBatchFiles\. 

Also the script details can be found in task scheduler by clicking on respective client folder and and then clicking on job to know the script related details as shown below in Actions tab:



Folders

Task Scheduler Folder / Client ID

Operational Notes & Script Target Location

 









AXIA

Legacy / Active client automation stream script located in C:\AutoMoveBatchFiles\

CMM

Active client automation stream script located in C:\AutoMoveBatchFiles\

GALEN

Active client automation stream script located in C:\AutoMoveBatchFiles\

GIA

Active client automation stream script located in C:\AutoMoveBatchFiles\. 

Uses Method C (Two-Script Fetching & Local Staging wipe).

PDWD

Platinum Derm - Uses Method B (Python Zip Extraction + Processing script).

PHMG

Palomar Health - Uses Method A (Standalone Single-Pass Network Move script).

RVHT

Active client automation stream script located in C:\AutoMoveBatchFiles\

Uses Method C (Two-Script Fetching & Local Staging wipe).

SKY

Skylakes - Uses Method C (Two-Script Fetching & Local Staging wipe).

THC

Active client automation stream script located in C:\AutoMoveBatchFiles\

Uses Method C (Two-Script Fetching & Local Staging wipe).

WWMG

Active client automation stream script located in C:\AutoMoveBatchFiles\

Uses Method C (Two-Script Fetching & Local Staging wipe).







 

4. Step-by-Step Setup Procedure

Follow these exact sequential phases to onboard the new client successfully.

Step 1: Check the Content Separator

Open a sample data file from the client using Notepad. Look at how the loops are split:

Asterisk Separation: If lines contain ST*835, you must use ST\*835 inside your script search commands. The backslash tells Windows to search for a literal asterisk instead of treating it as a wildcard.

Pipe Separation: If lines contain ST|835, change the script search commands to use ST|835.

Step 2: Create the Script File

Copy the code block from Section 4 that corresponds to your chosen processing method. Open a new Notepad file, paste the template, and edit the paths at the very top. Always save your finalized production scripts inside the standard server directory: C:\AutoMoveBatchFiles\.

Step 3: Perform a Manual Test Run

Place 2 or 3 test files inside the client's inbound folder. Navigate to C:\AutoMoveBatchFiles\ and double-click the script to run it manually. Confirm two outcomes:

The files leave the source folder and arrive in C:\Rhapsodyprod\AllClients_835_837\Input\ with the correct shorthand client prefix attached to the front of the filename.

A daily CSV spreadsheet log updates inside the designated logging path, capturing accurate file counts and sizes.

Step 4: Configure the Task Scheduler

Open the Windows Task Scheduler console on the server and construct a new daily task:

Security Settings: Select "Run whether user is logged on or not" and check "Run with highest privileges". Assign the task to an administrative service profile that has permissions to modify local disks and network UNC paths.

Action Settings: Choose "Start a program" and browse to your script. You must paste C:\AutoMoveBatchFiles directly into the "Start in (optional)" field. Leaving this blank will cause the script to fail when executing in the background.

 



5. Code Templates

Template A: Direct Move Script (PHMG Style)

@echo off

setlocal enabledelayedexpansion



:: ============================================================

:: CONFIGURATION

:: ============================================================

set "source_folder=\\10.2.1.27\YourClient\Inbound"

set "destination_folder=C:\Rhapsodyprod\AllClients_835_837\Input"

set "sftp_archive_folder=\\10.2.1.27\YourClient\Archive"

set "log_path=C:\Rhapsodyprod\Log_files\yourclient"

set "client_id=NEWCLIENT"

set "prefix=NEWCLIENT_"



:: ============================================================

:: AUTOMATIC LOGIC

:: ============================================================

for /f "tokens=2 delims==" %%I in ('"wmic os get localdatetime /value"') do set datetime=%%I

set "date=%datetime:~4,2%-%datetime:~6,2%-%datetime:~0,4%"

set "log_file=%log_path%\%date%_report_log_%client_id%.csv"

set "time_now=%time:~0,8%"



if not exist "%destination_folder%" mkdir "%destination_folder%"

if not exist "%sftp_archive_folder%\%date%" mkdir "%sftp_archive_folder%\%date%"

if not exist "%log_path%" mkdir "%log_path%"



set "archive_path=%sftp_archive_folder%\%date%"

set received_count=0

set pushed_count=0

set total_size_kb_rec=0

set total_size_kb_dest=0

set st835_count=0

set st837_count=0



if not exist "%log_file%" (

    echo FileName,Status,Size ^(KB^),TimeStamp,SourcePath,DestinationPath,835_Count,837_Count > "%log_file%"

)



for /r "%source_folder%" %%f in (*.*) do (

    set "source_file_path=%%f"

    set "filename=%%~nxf"

    set "newname=%prefix%!filename!"

    set "dest_file_path=%destination_folder%\!newname!"



    for %%A in ("!source_file_path!") do set /a size_kb_rec=%%~zA / 1024

    set "is_835=0"

    set "is_837=0"



    findstr /i "ST\*835" "%%f" >nul

    if !errorlevel! equ 0 ( set /a st835_count+=1 & set "is_835=1" )

    findstr /i "ST\*837" "%%f" >nul

    if !errorlevel! equ 0 ( set /a st837_count+=1 & set "is_837=1" )



    copy /Y "%%f" "%archive_path%" >nul

    echo !filename!,Received,!size_kb_rec!,%date% %time_now%,"%%f","%archive_path%\!filename!",!is_835!,!is_837! >> "%log_file%"

    set /a received_count+=1

    set /a total_size_kb_rec+=!size_kb_rec!



    move /Y "%%f" "!dest_file_path!" >nul



    for %%B in ("!dest_file_path!") do set /a size_kb_dest=%%~zB / 1024

    echo !newname!,Pushed,!size_kb_dest!,%date% %time_now%,"%%f","!dest_file_path!",!is_835!,!is_837! >> "%log_file%"

    set /a pushed_count+=1

    set /a total_size_kb_dest+=!size_kb_dest!

)



echo Summary for %date% (Daily Report) >> "%log_file%"

echo Files Received,!received_count! >> "%log_file%"

echo Files Pushed,!pushed_count! >> "%log_file%"

echo Total Size Received (KB),!total_size_kb_rec! >> "%log_file%"

echo Total Size Pushed (KB),!total_size_kb_dest! >> "%log_file%"

echo 835 Count,!st835_count! >> "%log_file%"

echo 837 Count,!st837_count! >> "%log_file%"



echo Summary for GCP >> "%log_file%"

echo %client_id%,!received_count!,%date% %time_now%,SFTP,Rhapsody,!st835_count!,!st837_count! >> "%log_file%"

endlocal

Template B: Zip Extraction Scripts (PDWD Style)

Step 1: Python Unzipper Utility (Save as C:\AutoMoveBatchFiles\Unzip_YourClient.py)

import os

import zipfile



def unzip_and_delete(source_folder, output_folder):

    if not os.path.exists(output_folder):

        os.makedirs(output_folder)



    for filename in os.listdir(source_folder):

        if filename.endswith(".zip"):

            file_path = os.path.join(source_folder, filename)

            try:

                with zipfile.ZipFile(file_path, 'r') as zip_ref:

                    zip_ref.extractall(output_folder)

                os.remove(file_path)

            except Exception as e:

                print(f"Error processing {filename}: {e}")



if __name__ == "__main__":

    unzip_and_delete(source_folder=r"\\10.2.1.27\YourClient\Inbound\837", output_folder=r"\\10.2.1.27\YourClient\Inbound\837\Extracts")

    unzip_and_delete(source_folder=r"\\10.2.1.27\YourClient\Inbound\835", output_folder=r"\\10.2.1.27\YourClient\Inbound\835\Extracts")

Step 2: File Processing Script

Deploy Template A immediately after the Python unzipper task completes to move, check, and log the unzipped file contents.

Template C: Local Staging Scripts (SKY Style)

Script 1: Fetch and Stage Files (Save as C:\AutoMoveBatchFiles\Fetch_YourClient.bat)

@echo off

set "source_folder=\\10.2.1.27\YourClient\Inbound"

set "destination_folder=C:\YOURCLIENT_StagingFiles"

set "archive_folder=C:\AutoMoveBatchFiles\Clientwise_Archive\YOURCLIENT_Archive"

set "sftp_archive_folder=\\10.2.1.27\YourClient\Archive"



for /f "tokens=2 delims==" %%I in ('"wmic os get localdatetime /value"') do set datetime=%%I

set date=%datetime:~4,2%-%datetime:~6,2%-%datetime:~0,4%



if not exist "%archive_folder%\%date%" mkdir "%archive_folder%\%date%"

set "archive_path=%archive_folder%\%date%"



ROBOCOPY "%source_folder%" "\\10.2.1.27\YourClient\Inbound_temp" /E

ROBOCOPY "%source_folder%" "%archive_path%" /E

ROBOCOPY "%source_folder%" "%destination_folder%" /E /mov

ROBOCOPY "%archive_folder%" "%sftp_archive_folder%" /MOVE /E

Script 2: Local Processing and Cleanup (Save as C:\AutoMoveBatchFiles\Process_YourClient.bat)

@echo off

setlocal enabledelayedexpansion



set "source_folder=C:\YOURCLIENT_StagingFiles"

set "destination_folder=C:\Rhapsodyprod\AllClients_835_837\Input"

set "log_path=C:\Rhapsodyprod\Log_files\yourclient"

set "client_id=NEWCLIENT"

set "prefix=NEWCLIENT_"



for /f "tokens=2 delims==" %%I in ('"wmic OS Get localdatetime /value"') do set datetime=%%I

set "date_today=%datetime:~4,2%-%datetime:~6,2%-%datetime:~0,4%"

set "log_file=%log_path%\%date_today%_report_log_%client_id%.csv"

set "time_now=%time:~0,8%"



set received_count=0

set pushed_count=0

set total_size_kb_rec=0

set total_size_kb_dest=0

set st835_count=0

set st837_count=0



if not exist "%log_file%" (

    echo FileName,Status,Size ^(KB^),TimeStamp,SourcePath,DestinationPath,835_Count,837_Count > "%log_file%"

)

if not exist "%destination_folder%" mkdir "%destination_folder%"



for /r "%source_folder%" %%F in (*.*) do (

    set "source_file_path=%%F"

    set "filename=%%~nxF"

    set "new_name=%prefix%!filename!"

    set "destination_file_path=%destination_folder%\!new_name!"



    for %%A in ("!source_file_path!") do set /a size_kb_rec=%%~zA / 1024



    findstr /i "ST\*835" "!source_file_path!" >nul

    if !errorlevel! equ 0 ( set /a st835_count+=1 )

    findstr /i "ST\*837" "!source_file_path!" >nul

    if !errorlevel! equ 0 ( set /a st837_count+=1 )



    copy /y "!source_file_path!" "!destination_file_path!" > nul



    echo !filename!,Received,!size_kb_rec!,%date_today% %time_now%,!source_file_path!,!destination_file_path!,!st835_count!,!st837_count! >> "%log_file%"

    set /a received_count+=1

    set /a total_size_kb_rec+=!size_kb_rec!

)



for /r "%destination_folder%" %%F in (*.*) do (

    set "destination_file_path=%%F"

    set "filename=%%~nxF"

    for %%A in ("!destination_file_path!") do set /a size_kb_dest=%%~zA / 1024

    echo !filename!,Pushed,!size_kb_dest!,%date_today% %time_now%,%source_folder%,!destination_file_path!,!st835_count!,!st837_count! >> "%log_file%"

    set /a pushed_count+=1

    set /a total_size_kb_dest+=!size_kb_dest!

)



echo Summary for %date_today% (Daily Report) >> "%log_file%"

echo Files Received,!received_count! >> "%log_file%"

echo Files Pushed,!pushed_count! >> "%log_file%"

echo Total Size Received (KB),!total_size_kb_rec! >> "%log_file%"

echo Total Size Pushed (KB),!total_size_kb_dest! >> "%log_file%"

echo 835 Count,!st835_count! >> "%log_file%"

echo 837 Count,!st837_count! >> "%log_file%"



echo Summary for GCP >> "%log_file%"

echo %client_id%,!received_count!,%date_today% !time_now!,SFTP,Rhapsody,!st835_count!,!st837_count! >> "%log_file%"



del /Q "%source_folder%\*.*"

endlocal

 

5. Key Contacts



ROLE

NAME

EMAIL

NOTE

File Support

Ravish Khokle

ravish.khokle@ikshealth.com 

File Support

File Transfer Support

Jay Bhoyar

jay.bhoyar@ikshealth.com 

File Transfer Support 

Rhapsody Support

Vipin, Ashwini

vipin.kumar@ikshealth.com, ashwini.nadar@ikshealth.com 

Rhapsody Processing Support






Client Onboarding Documentation For Rhapsody: Step-by-Step Processing & Automation Guide

This guide provides a clear, step-by-step walkthrough for onboarding new clients onto our automated file ingestion system. Follow the steps sequentially to configure, test, and deploy daily processing scripts.

 

1. System Flow Overview 

The diagram below shows how insurance claim files travel from the client into our systems every day:

STAGE 1: Ingress

 

STAGE 2: Scan & Log

 

STAGE 3: Delivery

 

Client Drop-off

Files arrive on the SFTP share or network folder.

➔

Script Execution

Script reads files, counts 835/837 records, and updates the daily CSV report.

➔

Rhapsody Core

Files are renamed with a prefix and moved to the central input folder.



 

2. Selecting the Processing Method

Before creating a script, identify the file delivery style the client requires. Choose one of the three active methods below:

Method

Operational Steps

Active Reference

 

Method A: Direct Move

1. Read network folder directly

2. Archive file copies

3. Move files straight to Rhapsody

PHMG (Palomar)

Method B: Zip Extraction

1. Python utility extracts compressed .zip folders

2. Delete original zip containers

3. Batch script processes the loose extracts

PDWD (Platinum Derm)

Method C: Local Staging

1. Script 1 copies files to a local server folder

2. Script 2 counts records and routes to Rhapsody

3. Staging folder is wiped completely clean

SKY (Skylakes)



 

3. Server & Client Folder Inventory (Server 10.2.1.83)

All file transfer automation tasks are hosted on production server 10.2.1.83. Below is the complete directory inventory of active client folders mapped in Windows Task Scheduler. Their corresponding processing scripts reside under C:\AutoMoveBatchFiles\. 

Also the script details can be found in task scheduler by clicking on respective client folder and and then clicking on job to know the script related details as shown below in Actions tab:



Folders

Task Scheduler Folder / Client ID

Operational Notes & Script Target Location

 









AXIA

Legacy / Active client automation stream script located in C:\AutoMoveBatchFiles\

CMM

Active client automation stream script located in C:\AutoMoveBatchFiles\

GALEN

Active client automation stream script located in C:\AutoMoveBatchFiles\

GIA

Active client automation stream script located in C:\AutoMoveBatchFiles\. 

Uses Method C (Two-Script Fetching & Local Staging wipe).

PDWD

Platinum Derm - Uses Method B (Python Zip Extraction + Processing script).

PHMG

Palomar Health - Uses Method A (Standalone Single-Pass Network Move script).

RVHT

Active client automation stream script located in C:\AutoMoveBatchFiles\

Uses Method C (Two-Script Fetching & Local Staging wipe).

SKY

Skylakes - Uses Method C (Two-Script Fetching & Local Staging wipe).

THC

Active client automation stream script located in C:\AutoMoveBatchFiles\

Uses Method C (Two-Script Fetching & Local Staging wipe).

WWMG

Active client automation stream script located in C:\AutoMoveBatchFiles\

Uses Method C (Two-Script Fetching & Local Staging wipe).







 

4. Step-by-Step Setup Procedure

Follow these exact sequential phases to onboard the new client successfully.

Step 1: Check the Content Separator

Open a sample data file from the client using Notepad. Look at how the loops are split:

Asterisk Separation: If lines contain ST*835, you must use ST\*835 inside your script search commands. The backslash tells Windows to search for a literal asterisk instead of treating it as a wildcard.

Pipe Separation: If lines contain ST|835, change the script search commands to use ST|835.

Step 2: Create the Script File

Copy the code block from Section 4 that corresponds to your chosen processing method. Open a new Notepad file, paste the template, and edit the paths at the very top. Always save your finalized production scripts inside the standard server directory: C:\AutoMoveBatchFiles\.

Step 3: Perform a Manual Test Run

Place 2 or 3 test files inside the client's inbound folder. Navigate to C:\AutoMoveBatchFiles\ and double-click the script to run it manually. Confirm two outcomes:

The files leave the source folder and arrive in C:\Rhapsodyprod\AllClients_835_837\Input\ with the correct shorthand client prefix attached to the front of the filename.

A daily CSV spreadsheet log updates inside the designated logging path, capturing accurate file counts and sizes.

Step 4: Configure the Task Scheduler

Open the Windows Task Scheduler console on the server and construct a new daily task:

Security Settings: Select "Run whether user is logged on or not" and check "Run with highest privileges". Assign the task to an administrative service profile that has permissions to modify local disks and network UNC paths.

Action Settings: Choose "Start a program" and browse to your script. You must paste C:\AutoMoveBatchFiles directly into the "Start in (optional)" field. Leaving this blank will cause the script to fail when executing in the background.

 



5. Code Templates

Template A: Direct Move Script (PHMG Style)

@echo off

setlocal enabledelayedexpansion



:: ============================================================

:: CONFIGURATION

:: ============================================================

set "source_folder=\\10.2.1.27\YourClient\Inbound"

set "destination_folder=C:\Rhapsodyprod\AllClients_835_837\Input"

set "sftp_archive_folder=\\10.2.1.27\YourClient\Archive"

set "log_path=C:\Rhapsodyprod\Log_files\yourclient"

set "client_id=NEWCLIENT"

set "prefix=NEWCLIENT_"



:: ============================================================

:: AUTOMATIC LOGIC

:: ============================================================

for /f "tokens=2 delims==" %%I in ('"wmic os get localdatetime /value"') do set datetime=%%I

set "date=%datetime:~4,2%-%datetime:~6,2%-%datetime:~0,4%"

set "log_file=%log_path%\%date%_report_log_%client_id%.csv"

set "time_now=%time:~0,8%"



if not exist "%destination_folder%" mkdir "%destination_folder%"

if not exist "%sftp_archive_folder%\%date%" mkdir "%sftp_archive_folder%\%date%"

if not exist "%log_path%" mkdir "%log_path%"



set "archive_path=%sftp_archive_folder%\%date%"

set received_count=0

set pushed_count=0

set total_size_kb_rec=0

set total_size_kb_dest=0

set st835_count=0

set st837_count=0



if not exist "%log_file%" (

    echo FileName,Status,Size ^(KB^),TimeStamp,SourcePath,DestinationPath,835_Count,837_Count > "%log_file%"

)



for /r "%source_folder%" %%f in (*.*) do (

    set "source_file_path=%%f"

    set "filename=%%~nxf"

    set "newname=%prefix%!filename!"

    set "dest_file_path=%destination_folder%\!newname!"



    for %%A in ("!source_file_path!") do set /a size_kb_rec=%%~zA / 1024

    set "is_835=0"

    set "is_837=0"



    findstr /i "ST\*835" "%%f" >nul

    if !errorlevel! equ 0 ( set /a st835_count+=1 & set "is_835=1" )

    findstr /i "ST\*837" "%%f" >nul

    if !errorlevel! equ 0 ( set /a st837_count+=1 & set "is_837=1" )



    copy /Y "%%f" "%archive_path%" >nul

    echo !filename!,Received,!size_kb_rec!,%date% %time_now%,"%%f","%archive_path%\!filename!",!is_835!,!is_837! >> "%log_file%"

    set /a received_count+=1

    set /a total_size_kb_rec+=!size_kb_rec!



    move /Y "%%f" "!dest_file_path!" >nul



    for %%B in ("!dest_file_path!") do set /a size_kb_dest=%%~zB / 1024

    echo !newname!,Pushed,!size_kb_dest!,%date% %time_now%,"%%f","!dest_file_path!",!is_835!,!is_837! >> "%log_file%"

    set /a pushed_count+=1

    set /a total_size_kb_dest+=!size_kb_dest!

)



echo Summary for %date% (Daily Report) >> "%log_file%"

echo Files Received,!received_count! >> "%log_file%"

echo Files Pushed,!pushed_count! >> "%log_file%"

echo Total Size Received (KB),!total_size_kb_rec! >> "%log_file%"

echo Total Size Pushed (KB),!total_size_kb_dest! >> "%log_file%"

echo 835 Count,!st835_count! >> "%log_file%"

echo 837 Count,!st837_count! >> "%log_file%"



echo Summary for GCP >> "%log_file%"

echo %client_id%,!received_count!,%date% %time_now%,SFTP,Rhapsody,!st835_count!,!st837_count! >> "%log_file%"

endlocal

Template B: Zip Extraction Scripts (PDWD Style)

Step 1: Python Unzipper Utility (Save as C:\AutoMoveBatchFiles\Unzip_YourClient.py)

import os

import zipfile



def unzip_and_delete(source_folder, output_folder):

    if not os.path.exists(output_folder):

        os.makedirs(output_folder)



    for filename in os.listdir(source_folder):

        if filename.endswith(".zip"):

            file_path = os.path.join(source_folder, filename)

            try:

                with zipfile.ZipFile(file_path, 'r') as zip_ref:

                    zip_ref.extractall(output_folder)

                os.remove(file_path)

            except Exception as e:

                print(f"Error processing {filename}: {e}")



if __name__ == "__main__":

    unzip_and_delete(source_folder=r"\\10.2.1.27\YourClient\Inbound\837", output_folder=r"\\10.2.1.27\YourClient\Inbound\837\Extracts")

    unzip_and_delete(source_folder=r"\\10.2.1.27\YourClient\Inbound\835", output_folder=r"\\10.2.1.27\YourClient\Inbound\835\Extracts")

Step 2: File Processing Script

Deploy Template A immediately after the Python unzipper task completes to move, check, and log the unzipped file contents.

Template C: Local Staging Scripts (SKY Style)

Script 1: Fetch and Stage Files (Save as C:\AutoMoveBatchFiles\Fetch_YourClient.bat)

@echo off

set "source_folder=\\10.2.1.27\YourClient\Inbound"

set "destination_folder=C:\YOURCLIENT_StagingFiles"

set "archive_folder=C:\AutoMoveBatchFiles\Clientwise_Archive\YOURCLIENT_Archive"

set "sftp_archive_folder=\\10.2.1.27\YourClient\Archive"



for /f "tokens=2 delims==" %%I in ('"wmic os get localdatetime /value"') do set datetime=%%I

set date=%datetime:~4,2%-%datetime:~6,2%-%datetime:~0,4%



if not exist "%archive_folder%\%date%" mkdir "%archive_folder%\%date%"

set "archive_path=%archive_folder%\%date%"



ROBOCOPY "%source_folder%" "\\10.2.1.27\YourClient\Inbound_temp" /E

ROBOCOPY "%source_folder%" "%archive_path%" /E

ROBOCOPY "%source_folder%" "%destination_folder%" /E /mov

ROBOCOPY "%archive_folder%" "%sftp_archive_folder%" /MOVE /E

Script 2: Local Processing and Cleanup (Save as C:\AutoMoveBatchFiles\Process_YourClient.bat)

@echo off

setlocal enabledelayedexpansion



set "source_folder=C:\YOURCLIENT_StagingFiles"

set "destination_folder=C:\Rhapsodyprod\AllClients_835_837\Input"

set "log_path=C:\Rhapsodyprod\Log_files\yourclient"

set "client_id=NEWCLIENT"

set "prefix=NEWCLIENT_"



for /f "tokens=2 delims==" %%I in ('"wmic OS Get localdatetime /value"') do set datetime=%%I

set "date_today=%datetime:~4,2%-%datetime:~6,2%-%datetime:~0,4%"

set "log_file=%log_path%\%date_today%_report_log_%client_id%.csv"

set "time_now=%time:~0,8%"



set received_count=0

set pushed_count=0

set total_size_kb_rec=0

set total_size_kb_dest=0

set st835_count=0

set st837_count=0



if not exist "%log_file%" (

    echo FileName,Status,Size ^(KB^),TimeStamp,SourcePath,DestinationPath,835_Count,837_Count > "%log_file%"

)

if not exist "%destination_folder%" mkdir "%destination_folder%"



for /r "%source_folder%" %%F in (*.*) do (

    set "source_file_path=%%F"

    set "filename=%%~nxF"

    set "new_name=%prefix%!filename!"

    set "destination_file_path=%destination_folder%\!new_name!"



    for %%A in ("!source_file_path!") do set /a size_kb_rec=%%~zA / 1024



    findstr /i "ST\*835" "!source_file_path!" >nul

    if !errorlevel! equ 0 ( set /a st835_count+=1 )

    findstr /i "ST\*837" "!source_file_path!" >nul

    if !errorlevel! equ 0 ( set /a st837_count+=1 )



    copy /y "!source_file_path!" "!destination_file_path!" > nul



    echo !filename!,Received,!size_kb_rec!,%date_today% %time_now%,!source_file_path!,!destination_file_path!,!st835_count!,!st837_count! >> "%log_file%"

    set /a received_count+=1

    set /a total_size_kb_rec+=!size_kb_rec!

)



for /r "%destination_folder%" %%F in (*.*) do (

    set "destination_file_path=%%F"

    set "filename=%%~nxF"

    for %%A in ("!destination_file_path!") do set /a size_kb_dest=%%~zA / 1024

    echo !filename!,Pushed,!size_kb_dest!,%date_today% %time_now%,%source_folder%,!destination_file_path!,!st835_count!,!st837_count! >> "%log_file%"

    set /a pushed_count+=1

    set /a total_size_kb_dest+=!size_kb_dest!

)



echo Summary for %date_today% (Daily Report) >> "%log_file%"

echo Files Received,!received_count! >> "%log_file%"

echo Files Pushed,!pushed_count! >> "%log_file%"

echo Total Size Received (KB),!total_size_kb_rec! >> "%log_file%"

echo Total Size Pushed (KB),!total_size_kb_dest! >> "%log_file%"

echo 835 Count,!st835_count! >> "%log_file%"

echo 837 Count,!st837_count! >> "%log_file%"



echo Summary for GCP >> "%log_file%"

echo %client_id%,!received_count!,%date_today% !time_now!,SFTP,Rhapsody,!st835_count!,!st837_count! >> "%log_file%"



del /Q "%source_folder%\*.*"

endlocal

 

5. Key Contacts



ROLE

NAME

EMAIL

NOTE

File Support

Ravish Khokle

ravish.khokle@ikshealth.com 

File Support

File Transfer Support

Jay Bhoyar

jay.bhoyar@ikshealth.com 

File Transfer Support 

Rhapsody Support

Vipin, Ashwini

vipin.kumar@ikshealth.com, ashwini.nadar@ikshealth.com 

Rhapsody Processing Support






Client Onboarding Documentation For Rhapsody: Step-by-Step Processing & Automation Guide

This guide provides a clear, step-by-step walkthrough for onboarding new clients onto our automated file ingestion system. Follow the steps sequentially to configure, test, and deploy daily processing scripts.

 

1. System Flow Overview 

The diagram below shows how insurance claim files travel from the client into our systems every day:

STAGE 1: Ingress

 

STAGE 2: Scan & Log

 

STAGE 3: Delivery

 

Client Drop-off

Files arrive on the SFTP share or network folder.

➔

Script Execution

Script reads files, counts 835/837 records, and updates the daily CSV report.

➔

Rhapsody Core

Files are renamed with a prefix and moved to the central input folder.



 

2. Selecting the Processing Method

Before creating a script, identify the file delivery style the client requires. Choose one of the three active methods below:

Method

Operational Steps

Active Reference

 

Method A: Direct Move

1. Read network folder directly

2. Archive file copies

3. Move files straight to Rhapsody

PHMG (Palomar)

Method B: Zip Extraction

1. Python utility extracts compressed .zip folders

2. Delete original zip containers

3. Batch script processes the loose extracts

PDWD (Platinum Derm)

Method C: Local Staging

1. Script 1 copies files to a local server folder

2. Script 2 counts records and routes to Rhapsody

3. Staging folder is wiped completely clean

SKY (Skylakes)



 

3. Server & Client Folder Inventory (Server 10.2.1.83)

All file transfer automation tasks are hosted on production server 10.2.1.83. Below is the complete directory inventory of active client folders mapped in Windows Task Scheduler. Their corresponding processing scripts reside under C:\AutoMoveBatchFiles\. 

Also the script details can be found in task scheduler by clicking on respective client folder and and then clicking on job to know the script related details as shown below in Actions tab:



Folders

Task Scheduler Folder / Client ID

Operational Notes & Script Target Location

 









AXIA

Legacy / Active client automation stream script located in C:\AutoMoveBatchFiles\

CMM

Active client automation stream script located in C:\AutoMoveBatchFiles\

GALEN

Active client automation stream script located in C:\AutoMoveBatchFiles\

GIA

Active client automation stream script located in C:\AutoMoveBatchFiles\. 

Uses Method C (Two-Script Fetching & Local Staging wipe).

PDWD

Platinum Derm - Uses Method B (Python Zip Extraction + Processing script).

PHMG

Palomar Health - Uses Method A (Standalone Single-Pass Network Move script).

RVHT

Active client automation stream script located in C:\AutoMoveBatchFiles\

Uses Method C (Two-Script Fetching & Local Staging wipe).

SKY

Skylakes - Uses Method C (Two-Script Fetching & Local Staging wipe).

THC

Active client automation stream script located in C:\AutoMoveBatchFiles\

Uses Method C (Two-Script Fetching & Local Staging wipe).

WWMG

Active client automation stream script located in C:\AutoMoveBatchFiles\

Uses Method C (Two-Script Fetching & Local Staging wipe).







 

4. Step-by-Step Setup Procedure

Follow these exact sequential phases to onboard the new client successfully.

Step 1: Check the Content Separator

Open a sample data file from the client using Notepad. Look at how the loops are split:

Asterisk Separation: If lines contain ST*835, you must use ST\*835 inside your script search commands. The backslash tells Windows to search for a literal asterisk instead of treating it as a wildcard.

Pipe Separation: If lines contain ST|835, change the script search commands to use ST|835.

Step 2: Create the Script File

Copy the code block from Section 4 that corresponds to your chosen processing method. Open a new Notepad file, paste the template, and edit the paths at the very top. Always save your finalized production scripts inside the standard server directory: C:\AutoMoveBatchFiles\.

Step 3: Perform a Manual Test Run

Place 2 or 3 test files inside the client's inbound folder. Navigate to C:\AutoMoveBatchFiles\ and double-click the script to run it manually. Confirm two outcomes:

The files leave the source folder and arrive in C:\Rhapsodyprod\AllClients_835_837\Input\ with the correct shorthand client prefix attached to the front of the filename.

A daily CSV spreadsheet log updates inside the designated logging path, capturing accurate file counts and sizes.

Step 4: Configure the Task Scheduler

Open the Windows Task Scheduler console on the server and construct a new daily task:

Security Settings: Select "Run whether user is logged on or not" and check "Run with highest privileges". Assign the task to an administrative service profile that has permissions to modify local disks and network UNC paths.

Action Settings: Choose "Start a program" and browse to your script. You must paste C:\AutoMoveBatchFiles directly into the "Start in (optional)" field. Leaving this blank will cause the script to fail when executing in the background.

 



5. Code Templates

Template A: Direct Move Script (PHMG Style)

@echo off

setlocal enabledelayedexpansion



:: ============================================================

:: CONFIGURATION

:: ============================================================

set "source_folder=\\10.2.1.27\YourClient\Inbound"

set "destination_folder=C:\Rhapsodyprod\AllClients_835_837\Input"

set "sftp_archive_folder=\\10.2.1.27\YourClient\Archive"

set "log_path=C:\Rhapsodyprod\Log_files\yourclient"

set "client_id=NEWCLIENT"

set "prefix=NEWCLIENT_"



:: ============================================================

:: AUTOMATIC LOGIC

:: ============================================================

for /f "tokens=2 delims==" %%I in ('"wmic os get localdatetime /value"') do set datetime=%%I

set "date=%datetime:~4,2%-%datetime:~6,2%-%datetime:~0,4%"

set "log_file=%log_path%\%date%_report_log_%client_id%.csv"

set "time_now=%time:~0,8%"



if not exist "%destination_folder%" mkdir "%destination_folder%"

if not exist "%sftp_archive_folder%\%date%" mkdir "%sftp_archive_folder%\%date%"

if not exist "%log_path%" mkdir "%log_path%"



set "archive_path=%sftp_archive_folder%\%date%"

set received_count=0

set pushed_count=0

set total_size_kb_rec=0

set total_size_kb_dest=0

set st835_count=0

set st837_count=0



if not exist "%log_file%" (

    echo FileName,Status,Size ^(KB^),TimeStamp,SourcePath,DestinationPath,835_Count,837_Count > "%log_file%"

)



for /r "%source_folder%" %%f in (*.*) do (

    set "source_file_path=%%f"

    set "filename=%%~nxf"

    set "newname=%prefix%!filename!"

    set "dest_file_path=%destination_folder%\!newname!"



    for %%A in ("!source_file_path!") do set /a size_kb_rec=%%~zA / 1024

    set "is_835=0"

    set "is_837=0"



    findstr /i "ST\*835" "%%f" >nul

    if !errorlevel! equ 0 ( set /a st835_count+=1 & set "is_835=1" )

    findstr /i "ST\*837" "%%f" >nul

    if !errorlevel! equ 0 ( set /a st837_count+=1 & set "is_837=1" )



    copy /Y "%%f" "%archive_path%" >nul

    echo !filename!,Received,!size_kb_rec!,%date% %time_now%,"%%f","%archive_path%\!filename!",!is_835!,!is_837! >> "%log_file%"

    set /a received_count+=1

    set /a total_size_kb_rec+=!size_kb_rec!



    move /Y "%%f" "!dest_file_path!" >nul



    for %%B in ("!dest_file_path!") do set /a size_kb_dest=%%~zB / 1024

    echo !newname!,Pushed,!size_kb_dest!,%date% %time_now%,"%%f","!dest_file_path!",!is_835!,!is_837! >> "%log_file%"

    set /a pushed_count+=1

    set /a total_size_kb_dest+=!size_kb_dest!

)



echo Summary for %date% (Daily Report) >> "%log_file%"

echo Files Received,!received_count! >> "%log_file%"

echo Files Pushed,!pushed_count! >> "%log_file%"

echo Total Size Received (KB),!total_size_kb_rec! >> "%log_file%"

echo Total Size Pushed (KB),!total_size_kb_dest! >> "%log_file%"

echo 835 Count,!st835_count! >> "%log_file%"

echo 837 Count,!st837_count! >> "%log_file%"



echo Summary for GCP >> "%log_file%"

echo %client_id%,!received_count!,%date% %time_now%,SFTP,Rhapsody,!st835_count!,!st837_count! >> "%log_file%"

endlocal

Template B: Zip Extraction Scripts (PDWD Style)

Step 1: Python Unzipper Utility (Save as C:\AutoMoveBatchFiles\Unzip_YourClient.py)

import os

import zipfile



def unzip_and_delete(source_folder, output_folder):

    if not os.path.exists(output_folder):

        os.makedirs(output_folder)



    for filename in os.listdir(source_folder):

        if filename.endswith(".zip"):

            file_path = os.path.join(source_folder, filename)

            try:

                with zipfile.ZipFile(file_path, 'r') as zip_ref:

                    zip_ref.extractall(output_folder)

                os.remove(file_path)

            except Exception as e:

                print(f"Error processing {filename}: {e}")



if __name__ == "__main__":

    unzip_and_delete(source_folder=r"\\10.2.1.27\YourClient\Inbound\837", output_folder=r"\\10.2.1.27\YourClient\Inbound\837\Extracts")

    unzip_and_delete(source_folder=r"\\10.2.1.27\YourClient\Inbound\835", output_folder=r"\\10.2.1.27\YourClient\Inbound\835\Extracts")

Step 2: File Processing Script

Deploy Template A immediately after the Python unzipper task completes to move, check, and log the unzipped file contents.

Template C: Local Staging Scripts (SKY Style)

Script 1: Fetch and Stage Files (Save as C:\AutoMoveBatchFiles\Fetch_YourClient.bat)

@echo off

set "source_folder=\\10.2.1.27\YourClient\Inbound"

set "destination_folder=C:\YOURCLIENT_StagingFiles"

set "archive_folder=C:\AutoMoveBatchFiles\Clientwise_Archive\YOURCLIENT_Archive"

set "sftp_archive_folder=\\10.2.1.27\YourClient\Archive"



for /f "tokens=2 delims==" %%I in ('"wmic os get localdatetime /value"') do set datetime=%%I

set date=%datetime:~4,2%-%datetime:~6,2%-%datetime:~0,4%



if not exist "%archive_folder%\%date%" mkdir "%archive_folder%\%date%"

set "archive_path=%archive_folder%\%date%"



ROBOCOPY "%source_folder%" "\\10.2.1.27\YourClient\Inbound_temp" /E

ROBOCOPY "%source_folder%" "%archive_path%" /E

ROBOCOPY "%source_folder%" "%destination_folder%" /E /mov

ROBOCOPY "%archive_folder%" "%sftp_archive_folder%" /MOVE /E

Script 2: Local Processing and Cleanup (Save as C:\AutoMoveBatchFiles\Process_YourClient.bat)

@echo off

setlocal enabledelayedexpansion



set "source_folder=C:\YOURCLIENT_StagingFiles"

set "destination_folder=C:\Rhapsodyprod\AllClients_835_837\Input"

set "log_path=C:\Rhapsodyprod\Log_files\yourclient"

set "client_id=NEWCLIENT"

set "prefix=NEWCLIENT_"



for /f "tokens=2 delims==" %%I in ('"wmic OS Get localdatetime /value"') do set datetime=%%I

set "date_today=%datetime:~4,2%-%datetime:~6,2%-%datetime:~0,4%"

set "log_file=%log_path%\%date_today%_report_log_%client_id%.csv"

set "time_now=%time:~0,8%"



set received_count=0

set pushed_count=0

set total_size_kb_rec=0

set total_size_kb_dest=0

set st835_count=0

set st837_count=0



if not exist "%log_file%" (

    echo FileName,Status,Size ^(KB^),TimeStamp,SourcePath,DestinationPath,835_Count,837_Count > "%log_file%"

)

if not exist "%destination_folder%" mkdir "%destination_folder%"



for /r "%source_folder%" %%F in (*.*) do (

    set "source_file_path=%%F"

    set "filename=%%~nxF"

    set "new_name=%prefix%!filename!"

    set "destination_file_path=%destination_folder%\!new_name!"



    for %%A in ("!source_file_path!") do set /a size_kb_rec=%%~zA / 1024



    findstr /i "ST\*835" "!source_file_path!" >nul

    if !errorlevel! equ 0 ( set /a st835_count+=1 )

    findstr /i "ST\*837" "!source_file_path!" >nul

    if !errorlevel! equ 0 ( set /a st837_count+=1 )



    copy /y "!source_file_path!" "!destination_file_path!" > nul



    echo !filename!,Received,!size_kb_rec!,%date_today% %time_now%,!source_file_path!,!destination_file_path!,!st835_count!,!st837_count! >> "%log_file%"

    set /a received_count+=1

    set /a total_size_kb_rec+=!size_kb_rec!

)



for /r "%destination_folder%" %%F in (*.*) do (

    set "destination_file_path=%%F"

    set "filename=%%~nxF"

    for %%A in ("!destination_file_path!") do set /a size_kb_dest=%%~zA / 1024

    echo !filename!,Pushed,!size_kb_dest!,%date_today% %time_now%,%source_folder%,!destination_file_path!,!st835_count!,!st837_count! >> "%log_file%"

    set /a pushed_count+=1

    set /a total_size_kb_dest+=!size_kb_dest!

)



echo Summary for %date_today% (Daily Report) >> "%log_file%"

echo Files Received,!received_count! >> "%log_file%"

echo Files Pushed,!pushed_count! >> "%log_file%"

echo Total Size Received (KB),!total_size_kb_rec! >> "%log_file%"

echo Total Size Pushed (KB),!total_size_kb_dest! >> "%log_file%"

echo 835 Count,!st835_count! >> "%log_file%"

echo 837 Count,!st837_count! >> "%log_file%"



echo Summary for GCP >> "%log_file%"

echo %client_id%,!received_count!,%date_today% !time_now!,SFTP,Rhapsody,!st835_count!,!st837_count! >> "%log_file%"



del /Q "%source_folder%\*.*"

endlocal

 

5. Key Contacts



ROLE

NAME

EMAIL

NOTE

File Support

Ravish Khokle

ravish.khokle@ikshealth.com 

File Support

File Transfer Support

Jay Bhoyar

jay.bhoyar@ikshealth.com 

File Transfer Support 

Rhapsody Support

Vipin, Ashwini

vipin.kumar@ikshealth.com, ashwini.nadar@ikshealth.com 

Rhapsody Processing Support quick  






