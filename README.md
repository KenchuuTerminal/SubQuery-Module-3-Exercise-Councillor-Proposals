# What is SubQuery?

SubQuery powers the next generation of Polkadot dApps by allowing developers to extract, transform and query blockchain data in real time using GraphQL. In addition to this, SubQuery provides production quality hosting infrastructure to run these projects in.

# SubQuery Example - Account transfers

This subquery example indexes the amount transferred of each account and it is an example of a 1-many entity relationshp. In other words, one account can have many receiving addresses.

# SubQuery Example - Account transfers : Result
![alt text](https://github.com/TsuyuKenchuu/SubQuery-Module-3-Exercise-Account-Transfer/blob/master/SubQuery-M03-EX2.JPG?raw=true)
     
#### Install the SubQuery CLI

Install SubQuery CLI globally on your terminal by using NPM:

```
npm install -g @subql/cli
```

Run help to see available commands and usage provide by CLI
```
subql help
```

## Initialize the Councillor Proposals

The first step in creating a SubQuery project is to create a project with the following command:
```
subql init --starter Councillor-Proposals
```
Then you should see a folder with your project name has been created inside the directory, you can use this as the start point of your project. And the files should be identical as in the [Directory Structure](https://doc.subquery.network/directory_structure.html).

Last, under the project directory, run following command to install all the dependency.
```
yarn install / npm install
```
## Configure your project

In the starter package, we have provided a simple example of project configuration. You will be mainly working on the following files:

- The Manifest in `project.yaml`
```shell
specVersion: 0.0.1
description: ''
repository: ''
schema: ./schema.graphql
network:
  endpoint: wss://polkadot.api.onfinality.io/public-ws
  dictionary: https://api.subquery.network/sq/subquery/dictionary-polkadot
dataSources:
  - name: main
    kind: substrate/Runtime
    startBlock: 1
    mapping:
      handlers:
        - handler: handleCouncilProposedEvent
          kind: substrate/EventHandler
          filter:
            module: council
            method: Proposed
        - handler: handleCouncilVotedEvent
          kind: substrate/EventHandler
          filter:
            module: council
            method: Voted
```

- The GraphQL Schema in `schema.graphql`
```shell
type Proposal @entity {
  id: ID!
  index: String!
  account: String
  hash: String
  voteThreshold: String
  block: BigInt
  voteHistory: [VoteHistory] @derivedFrom(field: "proposalHash")
}

type VoteHistory @entity {
  id: ID!
  proposalHash: Proposal
  approvedVote: Boolean!
  councillor: Councillor
  votedYes: Int
  votedNo: Int
  block: Int
}

type Councillor @entity {
  id: ID!
  numberOfVotes: Int
  voteHistory: [VoteHistory] @derivedFrom(field: "councillor")
}
```

- The Mapping functions in `src/mappings/` directory
```shell
import { SubstrateEvent } from "@subql/types";
import { bool, Int } from "@polkadot/types";
import { Proposal, VoteHistory, Councillor } from "../types/models";

export async function handleCouncilProposedEvent(
  event: SubstrateEvent
): Promise<void> {
  const {
    event: {
      data: [accountId, proposal_index, proposal_hash, threshold],
    },
  } = event;
  const proposal = new Proposal(proposal_hash.toString());
  proposal.index = proposal_index.toString();
  proposal.account = accountId.toString();
  proposal.hash = proposal_hash.toString();
  proposal.voteThreshold = threshold.toString();
  proposal.block = event.block.block.header.number.toBigInt();
  await proposal.save();
}

export async function handleCouncilVotedEvent(
  event: SubstrateEvent
): Promise<void> {
  // logger.info(JSON.stringify(event.event.data));
  const {
    event: {
      data: [councilorId, proposal_hash, approved_vote, numberYes, numberNo],
    },
  } = event;

  await ensureCouncillor(councilorId.toString());
  // Retrieve the record by the accountID
  const voteHistory = new VoteHistory(
    `${event.block.block.header.number.toNumber()}-${event.idx}`
  );
  voteHistory.proposalHashId = proposal_hash.toString();
  voteHistory.approvedVote = (approved_vote as bool).valueOf();
  voteHistory.councillorId = councilorId.toString();
  voteHistory.votedYes = (numberYes as Int).toNumber();
  voteHistory.votedNo = (numberNo as Int).toNumber();
  voteHistory.block = event.block.block.header.number.toNumber();
  // logger.info(JSON.stringify(voteHistory));
  await voteHistory.save();
}

async function ensureCouncillor(accountId: string): Promise<void> {
  // ensure that our account entities exist
  let councillor = await Councillor.get(accountId);
  if (!councillor) {
    councillor = new Councillor(accountId);
    councillor.numberOfVotes = 0;
  }
  councillor.numberOfVotes += 1;
  await councillor.save();
}
```

For more information on how to write the SubQuery, 
check out our doc section on [Define the SubQuery](https://doc.subquery.network/define_a_subquery.html) 

#### Code generation

In order to index your SubQuery project, it is mandatory to build your project first.
Run this command under the project directory.

````
yarn codegen
````

## Build the project

In order to deploy your SubQuery project to our hosted service, it is mandatory to pack your configuration before upload.
Run pack command from root directory of your project will automatically generate a `your-project-name.tgz` file.

```
yarn build
```

## Indexing and Query

#### Run required systems in docker


Under the project directory run following command:

```
docker-compose pull && docker-compose up
```
#### Query the project

Open your browser and head to `http://localhost:3000`.

Finally, you should see a GraphQL playground is showing in the explorer and the schemas that ready to query.

For the `subql-starter` project, you can try to query with the following code to get a taste of how it works.

````graphql
{
  query{
    starterEntities(first:10){
      nodes{
        field1,
        field2,
        field3
      }
    }
  }
}
````
