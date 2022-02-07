# log-to-aws-s3
Apply Linux logs to S3


How we can send system operational logs to AWS S3.

1 - Create a file inside directory /etc/logrotate.d. Choose a friendly name which we can identify the application.
    

2 - We need to know exactly where the application logs are. After that follow the exemple bellow.


cd /etc/logrotate.d/
touch log-to-s3

# Copy follow content to log-to-s3 file.

```
/var/log/cron               #logs we want to send to S3, you can put how many you need.
/var/log/maillog            #logs we want to send to S3, ...
/var/log/messages           #logs we want to send to S3, ...
/var/log/secure             #logs we want to send to S3, ...
/var/log/spooler            #logs we want to send to S3, ...

{
    notifempty
    maxsize 1G
    rotate 10
    copytruncate
    dateext
    dateformat -%Y-%m-%d
    compress
    compresscmd /usr/bin/xz
    compressoptions -9
    compressext .xz
    missingok
    lastaction
      /opt/logbackp/logbackp /var/log  logs
    endscript
}

```


3 - After that create a shell script and change fields like we can see.

vi /opt/logbackup/logbackup.sh

Copy / paste / save 

```
#!/bin/bash
#
# Move compressed files (in the informed path) to S3

# Config TODO: move to external file
S3_BUCKET=<log-to-s3-bucket>
#------ if you are using a on-premise server, uncommented the follow lines -----------
#export AWS_ACCESS_KEY_ID=<PUT YOUR ACCESS KEY HERE>
#export AWS_SECRET_ACCESS_KEY=<PUT YOUR SECRET KEY HERE>
#export AWS_DEFAULT_REGION=us-east-1
--------------------------------------------------------------------------------------


#--------- NO NEED TO CHANGE BELOW THIS LINE ---------

# Functions
function parse_file_name {
    # parse date info from file name
    # only files ending with YYYYMMDD.(gz|bz2)* or YYYY-MM-DD.(gz|bz2)* are considered
    # params: $1 - file name to be parsed
    # returns: sets global vars: filename, year, month (as empty if pattern does not match)

    # file name samples:
    # sample1.20170102.gz
    # sample2.20170103.bz2
    # sample3.2017-01-03.gz
    # sample4.log-2017-01-03.gz
    # sample5.log.2017-01-03.bz2.enc
    # sample6-with-dashes_log.2017-01-03.bz2-enc
    #local pattern="(.*)[.-]([0-9]{4})-?([0-9]{2})-?([0-9]{2})(.*).(gz|bz2|xz).*"
    local pattern="(.*)[.-]([0-9]{4})-?([0-9]{2})-?([0-9]{2})(.*).(gz|bz2|xz).*"

    if [[ "$1" =~ $pattern ]]
    then
        filename=$(basename ${BASH_REMATCH[1]})
        year=${BASH_REMATCH[2]}
        month=${BASH_REMATCH[3]}
        day=${BASH_REMATCH[4]}
    else
        filename=''
        year=''
        month=''
        day=''
    fi
}


function _test {
    parse_file_name 'sample6-with-dashes_log.2017-01-03.bz2-enc'
    if [ $filename != 'sample6-with-dashes_log' -o $year != '2017' -o $month != '01' ]
    then
        echo "ERROR: $0 FAILED TO RUN BASH REGEX"
        exit 1
    fi
}

function usage {
    cat << EOF
Move compressed files in the informed path to a S3 bucket
Usage: $0 <path> [<appname>]
    <path>     path with the files that will be backed up - must be the first argument
    <appname>  optional - application name used to group files
    Only files ending with .(gz|bz2) are considered. If file ends with
    YYYYMMDD.(gz|bz2) or YYYY-MM-DD.(gz|bz2), date information is parsed and used
    to organize them in the S3 bucket.
    --help     print this message
    --test     test if bash regex works properly

The files will be MOVED to the s3 bucket with the following key pattern prefix*:
    s3://<bucket-name>/<appname>/<hostname>/<year>/<month>/<filename prefix>/
    * date information is parsed from the filename
EOF
    exit 0
}

#------------------------------

LOG_PATH=$1
APP_NAME=$2
LOG_THIS=$LOG_PATH/$(basename $0 .sh).log
HOSTNAME=`hostname -s`
AWS_CMD=$(which aws)
[ -n "$APP_NAME" ] && KEY_PREFIX=$APP_NAME/$HOSTNAME || KEY_PREFIX=$HOSTNAME

# check args and stuff
[ -z "$1" -o "$1" = "--help" ] && usage

if [ "$1" = "--test" ]
then
    _test
    exit $?
fi

[ ! -d "$1" ] && echo "ERROR: invalid path" && exit 1

[ ! -x "$AWS_CMD" ] && echo "ERROR: aws command not found" && exit 1

# test bash regex
# Why? Because bash prior to version 3 does not have the regex match operator '=~'
_test

# do what should be done
for file in $(ls $LOG_PATH/*.gz $LOG_PATH/*.bz2 $LOG_PATH/*.xz 2>/dev/null)
do
    parse_file_name $file
    if [ -n $filename ]
    then
        $AWS_CMD s3 mv $file s3://$S3_BUCKET/$KEY_PREFIX/$year/$month/$day/$filename/ >> $LOG_THIS
    fi
done
```
#chmod 755 /opt/logbackup/logbackup.sh

# bash -x ./logbackp /var/log/ messages
+ S3_BUCKET=<Bucket destiny>
+ LOG_PATH=/var/log/
+ APP_NAME=messages
++ basename ./logbackp .sh
+ LOG_THIS=/var/log//logbackp.log
++ hostname -s
+ HOSTNAME=ip-xxx-xxx-xxx-xxx
++ which aws
+ AWS_CMD=/usr/local/bin/aws
+ '[' -n messages ']'
+ KEY_PREFIX=messages/ip-xxx-xxx-xxx-xxx
+ '[' -z /var/log/ -o /var/log/ = --help ']'
+ '[' /var/log/ = --test ']'
+ '[' '!' -d /var/log/ ']'
+ '[' '!' -x /usr/local/bin/aws ']'
+ _test
+ parse_file_name sample6-with-dashes_log.2017-01-03.bz2-enc
+ local 'pattern=(.*)[.-]([0-9]{4})-?([0-9]{2})-?([0-9]{2})(.*).(gz|bz2|xz).*'
+ [[ sample6-with-dashes_log.2017-01-03.bz2-enc =~ (.*)[.-]([0-9]{4})-?([0-9]{2})-?([0-9]{2})(.*).(gz|bz2|xz).* ]]
++ basename sample6-with-dashes_log
+ filename=sample6-with-dashes_log
+ year=2017
+ month=01
+ day=03
+ '[' sample6-with-dashes_log '!=' sample6-with-dashes_log -o 2017 '!=' 2017 -o 01 '!=' 01 ']'
++ ls '/var/log//*.gz' '/var/log//*.bz2' '/var/log//*.xz'


After apply script we can see something like that.

![image](https://user-images.githubusercontent.com/25347806/152860869-f92c13e3-24b7-4e29-b6f6-43bb689ce36c.png)


Thanks!!!!!! 
:)
