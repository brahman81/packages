# SDF - packages
  
1.  [Adding the SDF stable repository to your system](adding-the-sdf-stable-repository-to-your-system.md)
2.  [Quickstart](quickstart.md)
3.  [Installing individual packages](installing-individual-packages.md)
4.  [Upgrading](upgrading.md)
5.  [Running Horizon in production](running-horizon-in-production.md)
6.  [Building Packages](building-packages.md)
7.  [Running a Full Validator](running-a-full-validator.md)
8.  [Publishing a History archive](publishing-a-history-archive.md)
9.  [Backfilling a History archive](backfilling-a-history-archive.md)
10. [Monitoring](monitoring.md)
11. [Testnet Reset](testnet-reset.md)

## Backfilling a history archive

Given the choice, it is best to configure the History archive prior to your nodes initial synch to the network, this way your validator or archiver's History is published as you join/synch to the network.

However, if you have not published an archive during the node's initial synch it is still possible to use the [stellar-archivist](https://github.com/stellar/go/tree/master/tools/stellar-archivist) command line tool to mirror, scan and repair existing archives.

Using the SDF package repositories you can install `stellar-archivist` by running `apt-get install stellar-archivist`

The steps required to create a History archive for an existing validator (ie: basic validator -> full validator) are straightforward:

 * stop your stellar-core instance (`systemctl stop stellar-core`)
 * configure a History archive for the new node

```
[HISTORY.local]
get="cp /mnt/xvdf/stellar-core-archive/node_001/{0} {1}"
put="cp {0} /mnt/xvdf/stellar-core-archive/node_001/{1}"
mkdir="mkdir -p /mnt/xvdf/stellar-core-archive/node_001/{0}"
```

 * run new-hist to create the local archive

`# sudo -u stellar stellar-core --conf /etc/stellar/stellar-core.cfg new-hist local`

This command creates the History archive structure:

```
# tree -a /mnt/xvdf/stellar-core-archive/
/mnt/xvdf/stellar-core-archive
└── node_001
    ├── history
    │   └── 00
    │       └── 00
    │           └── 00
    │               └── history-00000000.json
    └── .well-known
        └── stellar-history.json

6 directories, 2 file
```
 * start your stellar-core instance (`systemctl start stellar-core`)
 * allow your node to join the network and watch it start publishing a few checkpoints to the newly created archive

```
2019-04-25T12:30:43.275 GDUQJ [History INFO] Publishing 1 queued checkpoints [16895-16895]: Awaiting 0/0 prerequisites of: publish-000041ff
```

At this stage your archiver/validator is successfully publishing it's history, this enables other users to join the network using your archive but unfortunately won't allow them to `CATCHUP_COMPLETE=true` as the archive only has partial network history.

##### Complete History Archive

If you decide to publish a complete archive enabling users to join the network from the genesis ledger (ledger 1) to the latest ledger then it is possible to use `stellar-archivist` to add all missing history data to your partial archive as well as verifying the state and integrity of your archive.

```
# stellar-archivist scan file:///mnt/xvdf/stellar-core-archive/node_001
2019/04/25 11:42:51 Scanning checkpoint files in range: [0x0000003f, 0x0000417f]
2019/04/25 11:42:51 Checkpoint files scanned with 324 errors
2019/04/25 11:42:51 Archive: 3 history, 2 ledger, 2 transactions, 2 results, 2 scp
2019/04/25 11:42:51 Scanning all buckets, and those referenced by range
2019/04/25 11:42:51 Archive: 30 buckets total, 30 referenced
2019/04/25 11:42:51 Examining checkpoint files for gaps
2019/04/25 11:42:51 Examining buckets referenced by checkpoints
2019/04/25 11:42:51 Missing history (260): [0x0000003f-0x000040ff]
2019/04/25 11:42:51 Missing ledger (260): [0x0000003f-0x000040ff]
2019/04/25 11:42:51 Missing transactions (260): [0x0000003f-0x000040ff]
2019/04/25 11:42:51 Missing results (260): [0x0000003f-0x000040ff]
2019/04/25 11:42:51 No missing buckets referenced in range [0x0000003f, 0x0000417f]
2019/04/25 11:42:51 324 errors scanning checkpoints
```

As you can tell from the output of the `scan` command some history, ledger, transactions, results are missing from the local history archive

You can now repair the missing data using stellar-archivist's `repair` command and a known full archive such as the SDF public history archive

`# stellar-archivist repair http://history.stellar.org/prd/core-testnet/core_testnet_001/ file:///mnt/xvdf/stellar-core-archive/node_001/`

```
2019/04/25 11:50:15 repairing http://history.stellar.org/prd/core-testnet/core_testnet_001/ -> file:///mnt/xvdf/stellar-core-archive/node_001/
2019/04/25 11:50:15 Starting scan for repair
2019/04/25 11:50:15 Scanning checkpoint files in range: [0x0000003f, 0x000041bf]
2019/04/25 11:50:15 Checkpoint files scanned with 244 errors
2019/04/25 11:50:15 Archive: 4 history, 3 ledger, 263 transactions, 61 results, 3 scp
2019/04/25 11:50:15 Error: 244 errors scanning checkpoints
2019/04/25 11:50:15 Examining checkpoint files for gaps
2019/04/25 11:50:15 Repairing history/00/00/00/history-0000003f.json
2019/04/25 11:50:15 Repairing history/00/00/00/history-0000007f.json
2019/04/25 11:50:15 Repairing history/00/00/00/history-000000bf.json
...
2019/04/25 11:50:22 Repairing ledger/00/00/00/ledger-0000003f.xdr.gz
2019/04/25 11:50:23 Repairing ledger/00/00/00/ledger-0000007f.xdr.gz
2019/04/25 11:50:23 Repairing ledger/00/00/00/ledger-000000bf.xdr.gz
...
2019/04/25 11:51:18 Repairing results/00/00/0e/results-00000ebf.xdr.gz
2019/04/25 11:51:18 Repairing results/00/00/0e/results-00000eff.xdr.gz
2019/04/25 11:51:19 Repairing results/00/00/0f/results-00000f3f.xdr.gz
...
2019/04/25 11:51:39 Repairing scp/00/00/00/scp-0000003f.xdr.gz
2019/04/25 11:51:39 Repairing scp/00/00/00/scp-0000007f.xdr.gz
2019/04/25 11:51:39 Repairing scp/00/00/00/scp-000000bf.xdr.gz
...
2019/04/25 11:51:50 Re-running checkpoing-file scan, for bucket repair
2019/04/25 11:51:50 Scanning checkpoint files in range: [0x0000003f, 0x000041bf]
2019/04/25 11:51:50 Checkpoint files scanned with 5 errors
2019/04/25 11:51:50 Archive: 264 history, 263 ledger, 263 transactions, 263 results, 241 scp
2019/04/25 11:51:50 Error: 5 errors scanning checkpoints
2019/04/25 11:51:50 Scanning all buckets, and those referenced by range
2019/04/25 11:51:50 Archive: 40 buckets total, 2478 referenced
2019/04/25 11:51:50 Examining buckets referenced by checkpoints
2019/04/25 11:51:50 Repairing bucket/57/18/d4/bucket-5718d412bdc19084dafeb7e1852cf06f454392df627e1ec056c8b756263a47f1.xdr.gz
2019/04/25 11:51:50 Repairing bucket/8a/a1/62/bucket-8aa1624cc44aa02609366fe6038ffc5309698d4ba8212ef9c0d89dc1f2c73033.xdr.gz
2019/04/25 11:51:50 Repairing bucket/30/82/6a/bucket-30826a8569cb6b178526ddba71b995c612128439f090f371b6bf70fe8cf7ec24.xdr.gz
...
```

A final scan of the local archive confirms that it has been successfully repaired

`# stellar-archivist scan file:///mnt/xvdf/stellar-core-archive/node_001`

```
2019/04/25 12:15:41 Scanning checkpoint files in range: [0x0000003f, 0x000041bf]
2019/04/25 12:15:41 Archive: 264 history, 263 ledger, 263 transactions, 263 results, 241 scp
2019/04/25 12:15:41 Scanning all buckets, and those referenced by range
2019/04/25 12:15:41 Archive: 2478 buckets total, 2478 referenced
2019/04/25 12:15:41 Examining checkpoint files for gaps
2019/04/25 12:15:41 Examining buckets referenced by checkpoints
2019/04/25 12:15:41 No checkpoint files missing in range [0x0000003f, 0x000041bf]
2019/04/25 12:15:41 No missing buckets referenced in range [0x0000003f, 0x000041bf]
```

  * start your stellar-core instance (`systemctl start stellar-core`)

You should now have a complete history archive being written to by your full validator
