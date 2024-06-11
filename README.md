import paramiko
import os
import schedule
import time
from datetime import datetime, timedelta

SFTP_HOST = 'Host IP'
SFTP_PORT = 22
SFTP_USERNAME = 'your username'
SFTP_PASSWORD = 'your password'
SFTP_REMOTE_PATH = '/home/path_to_your_files'  #the files which you want to transfer

LOCAL_DIRECTORY = '/home/path_of_remotefolder'

def transfer_files():
    try:
        transport = paramiko.Transport((SFTP_HOST, SFTP_PORT))
        transport.connect(username=SFTP_USERNAME, password=SFTP_PASSWORD)
        sftp = paramiko.SFTPClient.from_transport(transport)
        
        files_to_transfer = sorted([f for f in os.listdir(LOCAL_DIRECTORY) if os.path.isfile(os.path.join(LOCAL_DIRECTORY, f))])[-4:]
        
        for file_name in files_to_transfer:
            local_path = os.path.join(LOCAL_DIRECTORY, file_name)
            remote_path = os.path.join(SFTP_REMOTE_PATH, file_name)
            
            sftp.put(local_path, remote_path)
            print(f"Transferred {file_name} to {remote_path}")

            # transfer_time_file = open("transfer_time.txt", "a")
            # transfer_time_file.write(f"{file_name},{datetime.now()}\n")
            # transfer_time_file.close()

        sftp.close()
        transport.close()
        print("All files transferred successfully.")

        schedule.every().day.at("17:33").do(delete_old_files)  #add the time when you want to delete those transferred file(I wanted it transferred files to auto-delete after 30 days of its transfer)

    except Exception as e:
        print(f"An error occurred: {e}")

def delete_old_files():
    try:
        # transfer_time_file = open("transfer_time.txt", "r") 
        # lines = transfer_time_file.readlines()
        # transfer_time_file.close() (uncomment these lines, if you want a text file containing information about when and at what time files were transferred)
        ssh = paramiko.SSHClient()
        ssh.set_missing_host_key_policy(paramiko.AutoAddPolicy())
        ssh.connect(SFTP_HOST, SFTP_PORT, SFTP_USERNAME, SFTP_PASSWORD)
        sftp = ssh.open_sftp()
        
        current_time = datetime.now()
        files = sftp.listdir(SFTP_REMOTE_PATH)

        # for file_name in files:
        #     file_name, transfer_datetime_str = line.strip().split(',')
        #     transfer_datetime = datetime.strptime(transfer_datetime_str, "%Y-%m-%d %H:%M:%S.%f")
        #     if datetime.now() - transfer_datetime >= timedelta(days=30):
        #         file_path = os.path.join(SFTP_REMOTE_PATH, file_name)
        #         print(f'deleting_path: {file_path} {file_name}')
        
        for file_name in files:
            file_path = os.path.join(SFTP_REMOTE_PATH, file_name)
            file_attr = sftp.stat(file_path)
            file_mod_time = datetime.fromtimestamp(file_attr.st_mtime)
            if current_time - file_mod_time >= timedelta(days=30):  #(I wanted it transferred files to auto-delete after 30 days of its transfer)
                sftp.remove(file_path)
                print(f"Deleted {file_name} transferred on {file_mod_time}")
        sftp.close()
        ssh.close()
        # open("transfer_time.txt", "w").close()
        print("Old files deleted successfully.")

    except Exception as e:
        print(f"An error occurred while deleting old files: {e}")

schedule.every().day.at("17:35").do(transfer_files) #edit the timing as per your requirement)

while True:
    schedule.run_pending()
    time.sleep(1)
