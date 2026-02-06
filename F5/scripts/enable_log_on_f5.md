## Enable Log on F5
### Backup F5 config
> tmsh
> save /sys ucs [adc2].202602051638.ucs     # replace [adc2] with host name

### Test creating iRules in UAT paritiion using bash and tmsh
1. create the following testing file in windows 
> file name: test_irule_creation.irule
> 
> when HTTP_REQUEST {
>    # Log basic request information
>    log local0. "TEST IRULE: Client [IP::client_addr]:[TCP::client_port] -> [HTTP::host][HTTP::uri]"
> }

2. transfer file to F5
   
3. On F5, execute the following command
> 
> tmsh create ltm rule /UAT/irule-testing { rule { [string map {"\n" ""} [exec cat /tmp/test_irule_creation.irule]] } }
>

4. Goto F5 web GUI and see if the new iRule was created