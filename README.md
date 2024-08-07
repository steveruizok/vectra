# Vectra
Vectra is a local vector database for Node.js with features similar to [Pinecone](https://www.pinecone.io/) or [Qdrant](https://qdrant.tech/) but built using local files. Each Vectra index is a folder on disk. There's an `index.json` file in the folder that contains all the vectors for the index along with any indexed metadata.  When you create an index you can specify which metadata properties to index and only those fields will be stored in the `index.json` file. All of the other metadata for an item will be stored on disk in a separate file keyed by a GUID.

When queryng Vectra you'll be able to use the same subset of [Mongo DB query operators](https://www.mongodb.com/docs/manual/reference/operator/query/) that Pinecone supports and the results will be returned sorted by simularity. Every item in the index will first be filtered by metadata and then ranked for simularity. Even though every item is evaluated its all in memory so it should by nearly instantanious. Likely 1ms - 2ms for even a rather large index. Smaller indexes should be <1ms.

Keep in mind that your entire Vectra index is loaded into memory so it's not well suited for scenarios like long term chat bot memory. Use a real vector DB for that. Vectra is intended to be used in scenarios where you have a small corpus of mostly static data that you'd like to include in your prompt. Infinite few shot examples would be a great use case for Vectra or even just a single document you want to ask questions over.

Pinecone style namespaces aren't directly supported but you could easily mimic them by creating a separate Vectra index (and folder) for each namespace.

## Other Language Bindings
This repo contains the TypeScript/JavaScript binding for Vectra but other language bindings are being created. Since Vectra is file based, any language binding can be used to read or write a Vectra index. That means you can build a Vectra index using JS and then read it using Python.

- [vectra-py](https://github.com/BMS-geodev/vectra-py) - Python version of Vectra.

## Installation

```
$ npm install vectra
```

## Usage

First create an instance of `LocalIndex` with the path to the folder where you want you're items stored:

```typescript
import { LocalIndex } from 'vectra';

const index = new LocalIndex(path.join(__dirname, '..', 'index'));
```

Next, from inside an async function, create your index:

```typescript
if (!await index.isIndexCreated()) {
    await index.createIndex();
}
```

Add some items to your index:

```typescript
import { OpenAIApi, Configuration } from 'openai';

const configuration = new Configuration({
    apiKey: `<YOUR_KEY>`,
});

const api = new OpenAIApi(configuration);

async function getVector(text: string) {
    const response = await api.createEmbedding({
        'model': 'text-embedding-ada-002',
        'input': text,
    });
    return response.data.data[0].embedding;
}

async function addItem(text: string) {
    await index.insertItem({
        vector: await getVector(text),
        metadata: { text }
    });
}

// Add items
await addItem('apple');
await addItem('oranges');
await addItem('red');
await addItem('blue');
```

Then query for items:

```typescript
async function query(text: string) {
    const vector = await getVector(input);
    const results = await index.queryItems(vector, 3);
    if (results.length > 0) {
        for (const result of results) {
            console.log(`[${result.score}] ${result.item.metadata.text}`);
        }
    } else {
        console.log(`No results found.`);
    }
}

await query('green');
/*
[0.9036569942401076] blue
[0.8758153664568566] red
[0.8323828606103998] apple
*/

await query('banana');
/*
[0.9033128691220631] apple
[0.8493374123092652] oranges
[0.8415324469533297] blue
*/
```

## Creating an Index

You can create or modify an index with the `vectra CLI` located in the `bin` folder.

run `node ./bin/vectra.js` for all the available commands and options:

```bash
$ node ./bin/vectra.js
vectra <command>

Commands:
  vectra create <index>         create a new local index
  vectra delete <index>         delete an existing local index
  vectra add <index>            adds one or more web pages to an index
  vectra remove <index>         removes one or more documents from an index
  vectra stats <index>          prints the stats for a local index
  vectra query <index> <query>  queries a local index

Options:
  --version  Show version number                                       [boolean]
  --help     Show help                                                 [boolean]
```

### Example

To create a new index and add all web pages listed in a file `clouds.links` follow these steps:

- update one of the keys files in the `./indexes` folder. When working with Azur OpenAI, update `vectra.keys.azure-example`. For working OpenAI models, update `vectra.keys.openai-example`.

- run `node ./bin/vectra.js create <path to your index>`
- run `node ./bin/vectra.js add <path to your index> -l pages.links -k <path to your keys file>`

```bash
$ cat clouds.links
https://azure.com
https://openai.com

$ node ./bin/vectra.js create indexes/clouds
creating index at indexes/clouds

$ node ./bin/vectra.js add indexes/clouds -l clouds.links -k ./indexes/vectra.keys.azure-example
Adding Web Pages to Index
added https://azure.com
added https://openai.com
```
