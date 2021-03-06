# Testing offline-log-collector

NOTE These are not unit tests per se, but just wrapper around the actual script that sets up a test env first. 

This will generate a tarball for you that you can then ingest using the `offline-log-ingester` test script. 

However, it will not run TableAnalyzer as of right now, since it is not yet configured to allow custom nodetool commands. Since this test relies on ccm, and uses `ccm <ccm-hostname> nodetool` to run nodetool commands, TableAnalyzer would not be running against the test cluster.

### Setup Tests
- Requires python3 and pip3
- Make sure no other cassandra instance is running on your localhost (or else ccm will conflict with those ports, e.g., `OSError: [Errno 98] Address already in use`)

### Run Tests
Navigate to this folder (`cassandra-analyzer/offline-log-collector/`) in a terminal, then just run this:

```
python3 -m venv ./venv
source ./venv/bin/activate
# install project requirements
pip3 install -r requirements.txt

# install requirements for test
pip3 install -r test/requirements.txt

# run the test script
cd test
python3 collect_logs_test.py
```

Note:
- Make sure to use venv, so that when calling `ccm node1 nodetool` for example in NodeAnalyzer, it can work even though NodeAnalyzer runs as root user. 
- `config/settings.test.yaml` is expecting that the `venv` dir is found at `cassandra-analyzer/offline-log-collector/venv` and that the script is called from the `test` dir. Following the instructions as specified above will do this for you.
- In other words, make sure you follow the instructions above exactly as specified.

### What's Next?
You now have a tarball that you can test using `offline-log-ingester`. [Head over to instructions for `offline-log-ingester` to get started](../../offline-log-ingester/test/README.md).

### Debugging Tests
#### ERROR: `File exists: '$HOME/.ccm/test_cluster'`: 
    The test did not clean up correctly from last time (`test_cluster` is the name of the cluster we use for testing with ccm). The test script might have already removed the `test_cluster` for you, and you can just run the test again and it should work. 
    
    If not though, and **ASSUMING YOU DON'T NEED THAT CLUSTER ANYMORE:** Remove the old cluster so you can run test again: 
    ```
    ccm test_client remove

    # now run the test again
    python3 collect_logs_test.py
    ```

#### ERROR: `Command 'ccm' not found, but there are 20 similar ones.`
You can get this when the test is running NodeAnalyzer. Since it requires root access, it is running as root. If you did not use `venv`, you won't have access to `ccm` within root. 

You could try to modify the `test/config/settings.test.yaml` file to get it working, but better is just to install everything using venv as the instructions above specify.