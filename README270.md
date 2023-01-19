1. Created a new python virtual environment and installed stix-shifter and kestrel in that environment:
   ```
   virtualenv -p python3 huntingspace && source huntingspace/bin/activate
   python -m pip install --upgrade pip
   cd ../stix-shifter/
   INSTALL_REQUIREMENTS_ONLY=1 python3 setup.py install
   cd ../kestrel-lang/
   pip install -e .
   ```

2. Defined `~/.config/kestrel/stixshifter.yaml` configuration file with 2 profiles, one using the default and one using the beats dialect for elastic_ecs
   ```
   profiles:
       winlaptop141:
           connector: elastic_ecs
           connection:
               host: ***.**.***.ibm.com
               port: 9200
               selfSignedCert: false
               indices: winlogbeat-*
           config:
               auth:
                   id: ***
                   api_key: ***
       winlaptop141-beats:
           connector: elastic_ecs
           connection:
               host: ***.**.***.ibm.com
               port: 9200
               selfSignedCert: false
               indices: winlogbeat-*
               options:
                   dialects:
                     - beats
           config:
               auth:
                   id: ***
                   api_key: ***
   ```
3. Defined 2 hunting flows for the elastic_ecs connector.  `dialect-default-ecs.hf` uses the `default` dialect:
   ```
   ips = GET ipv4-addr FROM stixshifter://winlaptop141
         WHERE value = '127.0.0.1'
         START 2022-01-17T12:00:00Z STOP 2022-10-18T00:00:00Z
   ```
   `dialect-beats-ecs.hf` uses the `beats` dialect:
   ```
   ips = GET ipv4-addr FROM stixshifter://winlaptop141-beats
         WHERE value = '127.0.0.1'
         START 2022-01-17T12:00:00Z STOP 2022-10-18T00:00:00Z
   ```

4. Tried to run the 2 hunting flows, using `kestrel dialect-default-ecs.hf` and `kestrel dialect-beats-ecs.hf` commands.  When running the first command, the package `stix-shifter-modules-elastic-ecs==4.5.2` was installed, and the query completed successfully.
   ```
   (huntingspace) cmadam@DESKTOP:~$ kestrel dialect-default-ecs.hf
   Collecting stix-shifter-modules-elastic-ecs==4.5.2
     Downloading stix_shifter_modules_elastic_ecs-4.5.2-py2.py3-none-any.whl (58 kB)
       ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 58.9/58.9 kB 1.6 MB/s eta 0:00:00
    Installing collected packages: stix-shifter-modules-elastic-ecs
    Successfully installed stix-shifter-modules-elastic-ecs-4.5.2
    [SUMMARY] block executed in 18 seconds
    VARIABLE      TYPE  #(ENTITIES)  #(RECORDS)  directory*  file*  ipv4-addr*  ipv6-addr*  mac-addr*  network-traffic*  process*  user-account*  x-ecs-destination*  x-ecs-network*  x-ecs-source*  x-ecs-user*  x-oca-asset*  x-oca-event*
        ips ipv4-addr            1        6324          70   6324       18971        6324       6324              6324      6324           6322                4240            4240           6324         6322          6324          6324
    *Number of related records cached.
     ```
   The second command failed to execute:
   ```
   (huntingspace) cmadam@DESKTOP:~$ kestrel dialect-beats-ecs.hf
   Traceback (most recent call last):
   . . .
   File "/home/cmadam/kestrel-lang/src/kestrel_datasource_stixshifter/interface.py", line 172, in query
     raise DataSourceError(
   kestrel.exceptions.DataSourceError: [ERROR] DataSourceError: STIX-shifter translation from STIX to native query failed with message: elastic_ecs connector error => unknown dialect : beats
   please check data source config or test the query manually.
   ```

5. The `elastic-ecs` module installed was an old version that does not have the `beats` dialect.  To fix this problem, I have copied recursively the contents of the `elastic_ecs` module from the stix_shifter develop branch into the virtual workspace.
   ```
   (huntingspace) cmadam@DESKTOP:~$ cp -r stix-shifter/stix_shifter_modules/elastic_ecs/* kestrel-lang/huntingspace/lib/python3.8/site-packages/stix_shifter_modules/elastic_ecs/*
   ```
6. After making this change, both huntflows worked, and each flow invoked the correct dialect.  To make sure that the dialect was invoked correctly, I have printed the `dsl` variable returned in the `interface.py` file:
   ```
   (huntingspace) cmadam@DESKTOP:~$ kestrel dialect-beats-ecs.hf
   dsl = {'queries': ['(source.ip.keyword : "127.0.0.1" OR destination.ip.keyword : "127.0.0.1" OR client.ip : "127.0.0.1" OR server.ip : "127.0.0.1" OR host.ip.keyword : "127.0.0.1" OR dns.resolved_ip : "127.0.0.1") AND (@timestamp:["2022-01-17T12:00:00.000Z" TO "2022-10-18T00:00:00.000Z"])']}
   [SUMMARY] block executed in 18 seconds
   VARIABLE      TYPE  #(ENTITIES)  #(RECORDS)  directory*  file*  ipv4-addr*  ipv6-addr*  mac-addr*  network-traffic*  process*  user-account*  x-ecs-destination*  x-ecs-network*  x-ecs-source*  x-ecs-user*  x-oca-asset*  x-oca-event*
        ips ipv4-addr            1        6324          70   6324       18971        6324       6324              6324      6324           6322                4240            4240           6324         6322          6324          6324
	*Number of related records cached.
	
   (huntingspace) cmadam@DESKTOP:~$ kestrel dialect-default-ecs.hf
   dsl = {'queries': ['(source.ip : "127.0.0.1" OR destination.ip : "127.0.0.1" OR client.ip : "127.0.0.1" OR server.ip : "127.0.0.1" OR host.ip : "127.0.0.1" OR dns.resolved_ip : "127.0.0.1") AND (@timestamp:["2022-01-17T12:00:00.000Z" TO "2022-10-18T00:00:00.000Z"])']}
   [SUMMARY] block executed in 17 seconds
   VARIABLE      TYPE  #(ENTITIES)  #(RECORDS)  directory*  file*  ipv4-addr*  ipv6-addr*  mac-addr*  network-traffic*  process*  user-account*  x-ecs-destination*  x-ecs-network*  x-ecs-source*  x-ecs-user*  x-oca-asset*  x-oca-event*
        ips ipv4-addr            1        6324          70   6324       18971        6324       6324              6324      6324           6322                4240            4240           6324         6322          6324          6324
   *Number of related records cached.
   ```
7. I did not, however, see any difference between the results returned by the two dialects.  At least the number of records seems to be the same in both cases.