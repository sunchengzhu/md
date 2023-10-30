# Axon hardfork user guide

## Hardfork introduction

Currently, Axon has only one hardfork named `Andromeda`. The name of the hardfork is derived from one of the 88 constellations, and in the future, names will continue to be chosen based on the [order of the constellations](https://en.wikipedia.org/wiki/IAU_designated_constellations#List). After enabling `Andromeda`, [validators](https://github.com/axonweb3/axon/blob/f9974e62924693494476560316db9f70bc650b80/devtools/chain/nodes/node_1.toml#L3) are allowed to modify metadata, such as `max_contract_limit`, through [the system contract `0xffffffffffffffffffffffffffffffffffffff01`](https://docs.axonweb3.io/contract/system_contacts#metadata). Axon already has [hardfork test CI](https://github.com/axonweb3/axon/blob/f9974e62924693494476560316db9f70bc650b80/.github/workflows/hardfork_test.yml)  and [a test project available](https://github.com/axonweb3/axon-hardfork-test) for your reference.

## Usage steps

1. Build Axon from source code

   ```bash
   cd $your_workspace
   git clone https://github.com/axonweb3/axon.git
   cd axon
   cargo build
   ```

   

2. Start multiple Axon nodes

   [`reset.sh`](https://github.com/axonweb3/axon-hardfork-test/blob/5c9c172cc1ed1dff544f7e092f7052c314030c1d/reset.sh) is used to clear data and start axon nodes. You can also use it for the first-time startup.

   ```bash
   git clone https://github.com/axonweb3/axon-hardfork-test.git
   cd axon-hardfork-test
   bash reset.sh $your_workspace/axon
   ```

   One of Axon's default configurations is [`hardforks = []`](https://github.com/axonweb3/axon/blob/f9974e62924693494476560316db9f70bc650b80/devtools/chain/specs/multi_nodes/chain-spec.toml#L10), which defaults to enabling hardfork.  `reset.sh`  changes `hardforks = []` to `hardforks = ["None"]` to disable hardfork. Later on, hardfork can be enabled using the `hardfork -c` followed by specifying the `hardfork-start-number`.

    You should see an output similar to this following:

   ```
No process found listening on port 8001
   No process found listening on port 8002
   No process found listening on port 8003
   No process found listening on port 8004
   hardforks = ["None"]
   node_1 height: 6
   node_2 height: 6
   node_3 height: 6
   node_4 height: 6
   {
     "jsonrpc": "2.0",
     "result": {},
     "id": 1
   }
   {
     "jsonrpc": "2.0",
     "result": {},
     "id": 2
   }
   {
     "jsonrpc": "2.0",
     "result": {},
     "id": 3
   }
   {
     "jsonrpc": "2.0",
     "result": {},
     "id": 4
   }
   ```
   
   `height: 6` indicates that the nodes are producing blocks normally. Querying with `axon_getHardforkInfo` and receiving `"result": {}` implies that hardfork is disabled.

   

3. Enable hardfork

      `hardfork.sh` enables the hardfork by [default after 30 blocks](https://github.com/axonweb3/axon-hardfork-test/blob/5c9c172cc1ed1dff544f7e092f7052c314030c1d/hardfork.sh#L18).

       `hardfork.sh`  first kills all nodes. While the nodes are stopped, it executes `hardfork -c` to specify the `hardfork-start-number`. The nodes are then restarted. After startup, the status of `Andromeda` can be checked using `axon_getHardforkInfo`, and the expected status value should be `determined`.

      ```bash
      bash hardfork.sh $your_workspace/axon	
      ```

      You should see an output similar to this following:

      ```bash
      axon_path: /Users/sunchengzhu/tmp/axon
      hardfork-start-number: 694
      Killing processes on port 8001: 9285
      Killing processes on port 8002: 9286
      Killing processes on port 8003: 9287
      Killing processes on port 8004: 9288
      node_1 height: 670
      node_2 height: 670
      node_3 height: 670
      node_4 height: 670
      {
        "jsonrpc": "2.0",
        "result": {
          "Andromeda": "determined"
        },
        "id": 1
      }
      {
        "jsonrpc": "2.0",
        "result": {
          "Andromeda": "determined"
        },
        "id": 2
      }
      {
        "jsonrpc": "2.0",
        "result": {
          "Andromeda": "determined"
        },
        "id": 3
      }
      {
        "jsonrpc": "2.0",
        "result": {
          "Andromeda": "determined"
        },
        "id": 4
      }
      ```

      

4. Wait and check hardfork info

   You can use this following test case to verify if the status of `Andromeda` becomes `enabled` when the nodes reach the specified height.

   ```bash
   npm install
   npx hardhat test --grep "check hardfork info after hardfork"
   ```

   

5. Deploy a contract larger than default `max_contract_limit`

   The expected outcome of this test case is that the deployment transaction [returns an error message containing `CreateContractLimit`](https://github.com/axonweb3/axon-hardfork-test/blob/5c9c172cc1ed1dff544f7e092f7052c314030c1d/test/checkMetadata.ts#L18-L25).

   ```bash
   npx hardhat test --grep "deploy a big contract larger than max_contract_limit"
   ```

   

6. Increase `max_contract_limit`

   This test case increases the `max_contract_limit` by using the system contract `0xffffffffffffffffffffffffffffffffffffff01`.

   ```bash
   npx hardhat test --grep "update max_contract_limit"
   ```



7. Test if  new `max_contract_limit` is effective

   Deploy the previous contract again, with the expectation that it will deploy successfully.

   ```bash
   npx hardhat test --grep "deploy a big contract smaller than max_contract_limit"
   ```

   

