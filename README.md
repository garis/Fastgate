# Fastgate

All the firware are located here: http://59.0.121.191:8080/ACS-server/file/*FILENAME*
Where *FILENAME* is equal to: 
0.00.89_FW_200_Askey
0.00.81_FW_200_Askey
0.00.67_FW_200_Askey
0.00.47_FW_200_Askey
0.00.267_FW_200_Askey
0.00.167_FW_200_Askey

So, for example, to obtain the last firware just go to http://59.0.121.191:8080/ACS-server/file/0.00.89_FW_200_Askey

To exctract the files we need the utility in https://github.com/nlitsme/ubidump (this is the first one that worked for me).

Doing a binwalk on the file gives us:

$ binwalk 0.00.89_FW_200_Askey

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
94            0x5E            JFFS2 filesystem, little endian
3276894       0x32005E        UBI erase count header, version: 1, EC: 0x0, VID header offset: 0x800, data offset: 0x1000

We don't need the JFFS2 filesystem, only the UBI part. To do that we can cut the file using dd.
To do that we need the file dimension and the UBI starting point.

The file dimension can be retrieved using ls:

$ ls -la

total 55836
drwxr-xr-x  3 riccardo users     4096 set 24 10:07 .
drwx------ 83 riccardo users    12288 set 24 10:08 ..
-rw-r--r--  1 riccardo users **57147506** set 24 10:05 0.00.89_FW_200_Askey


The starting point is exctracted using binwalk:

$ binwalk 0.00.89_FW_200_Askey

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
94            0x5E            JFFS2 filesystem, little endian
**3276894**       0x32005E        UBI erase count header, version: 1, EC: 0x0, VID header offset: 0x800, data offset: 0x1000

To cut the file we invoke the mighty dd with the values that we have found:

dd if=0.00.89_FW_200_Askey of=0.00.89_FW_200_Askey.UBI bs=1 skip=**3276894** count=**57147506**

Now we can use ubidump (after some initial configuration depending on the system config):

git clone https://github.com/nlitsme/ubidump.git
sudo pip install crcmod
yay -S python-lzo
python ubidump/ubidump.py -s bump 0.00.81_FW_200_Askey.UBI

We now have the filesystem in bump/rootfs_ubifs/

Now we can proceed a by further in the analysis.

Looking around in the filesystem we can see that the web gui is managed by mini_httpd:

$ grep -i -r "mini_httpd"
etc/sniproxy/urlfilter3.sh:ln -sf /usr/sbin/mini_httpd $URLFILTER_WEBD_PATH 
etc/sniproxy/urlfilter3.sh:$URLFILTER_WEBD_PATH -d $URLFILTER_WEB_PATH -c "**.cgi" -u admin -S -E /web/mini_httpd.pem -p $webd_https_port 
bin/web.sh:#mv /tmp/mini_httpd /usr/sbin 
bin/web.sh:killall -9 mini_httpd 
**bin/web.sh:mini_httpd -d /web -c "STATUSAPI/**.cgi|STATUSAPI/*" -u admin -T "UTF-8" -p 8080**

And we can see that mini_httpd depends on a .cgi file:

$ grep -i -r "status.cgi"
web/factory/mtcmd/mt_info.htm:          "url":"/status.cgi", 
web/fw-router/app/app.js:.value('apiServiceUrl', '/status.cgi') 
**web/js/status.js:var CGI_NAME='status.cgi';**
web/js/status.js:       "url":"http://192.168.101.93/STATUSAPI/status.cgi",
web/js/status.js:       "url":"http://192.168.101.93/STATUSAPI/status.cgi",
web/js/status.js:       "url":http://192.168.101.93/STATUSAPI/status.cgi,

So, using Ghidra, we can start looking in the status.cgi.




