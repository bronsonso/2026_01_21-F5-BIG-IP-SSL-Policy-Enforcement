## Enable Log on F5
### Backup F5 config
> tmsh
> save /sys ucs [adc2].202602051638.ucs     # replace [adc2] with host name

### Test creating iRules in UAT paritiion using bash and tmsh

1. create the following testing file in windows 

` file name: test_irule_creation.irule </br>
` 
` when HTTP_REQUEST { </br>
`    # Log basic request information </br>
`    log local0. "TEST IRULE: Client [IP::client_addr]:[TCP::client_port] -> [HTTP::host][HTTP::uri]" </br>
` } </br>

1. check if iRule existed
> tmsh list ltm rule | grep -E "test.irule"

2. transfer file to F5 (using winscp)
  
3. On F5, execute the following command
> 
> tmsh create ltm rule /dev/test { rule { [string map {"\n" ""} [exec cat /tmp/test_irule_creation.irule]] } }

4. check irule syntax
> tmsh show ltm rule <irule_ame> syntax-check
> tmsh show ltm rule <irule_ame> syntax-check detail

4. Goto F5 web GUI and see if the new iRule was created

5. Remove the test.irule
> delete ltm rule test.irule

### Update iRule directly in file and upload back to server

1. Backup bigip.conf file in command line.
> cp /config/bigip.conf /config/bigip.conf.BAK

2. Download /config/bigip.conf to your pc.

3. Open bigip.conf file via text editor.

4. Use "find and replace" tool.

5. Upload bigip.conf file to /config folder.

6. Verify config.
> tmsh load sys config verify

7. If no error, load config.
> tmsh load sys config