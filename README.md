# Elastically, **Elastica** based framework

*This project is a work in progress.*

![Under Construction](https://jolicode.com/media/original/2019/construction.gif)

[![Build Status](https://travis-ci.org/jolicode/elastically.svg?branch=master)](https://travis-ci.org/jolicode/elastically)

**Feedback welcome!**

Opinionated [Elastica](https://github.com/ruflin/Elastica) based framework to bootstrap PHP and Elasticsearch implementations.

- DTO are first class citizen, you send object for documents, and get objects back, **like an ODM**;
- All indexes are versioned / aliased;
- Mappings are done in YAML;
- Analysis is separated from mappings;
- 100% compatibility with [ruflin/elastica](https://github.com/ruflin/Elastica);
- Designed for Elasticsearch 7+ (no types), compatible with both ES 6 and ES 7;
- Symfony Messenger Handler support (with or without spool);
- Symfony HttpClient compatible transport;
- Extra commands to monitor, update mapping, reindex... Commonly implemented tasks.

## Demo

Quick example of what the library do on top of Elastica:

```php
// Your own DTO, or one generated by Jane (see below)
class Beer
{
    public $foo;
    public $bar;
}

// Building the Index from a mapping config
use JoliCode\Elastically\Client;
use Elastica\Document;

// New Client object with new options
$client = new Client([
    // Where to find the mappings
    Client::CONFIG_MAPPINGS_DIRECTORY => __DIR__.'/mappings',
    // What object to find in each index
    Client::CONFIG_INDEX_CLASS_MAPPING => [
        'beers' => Beer::class,    
    ],
]);

// Class to build Indexes
$indexBuilder = $client->getIndexBuilder();

// Create the Index in Elasticsearch
$index = $indexBuilder->createIndex('beers');

// Set the proper aliases
$indexBuilder->markAsLive($index, 'beers');

// Class to index DTO in an Index
$indexer = $client->getIndexer();

$dto = new Beer();
$dto->bar = 'American Pale Ale';
$dto->foo = 'Hops from Alsace, France';

// Add a document to the queue
$indexer->scheduleIndex('beers', new Document('123', $dto));
$indexer->flush();

// Force index refresh if needed
$indexer->refresh('beers');

// Get the Document (new!)
$results = $client->getIndex('beers')->getDocument('123');

// Get the DTO (new!)
$results = $client->getIndex('beers')->getModel('123');

// Perform a search
$results = $client->getIndex('beers')->search('alsace');

// Get the Elastic Document
$results->getDocuments()[0];

// Get the Elastica compatible Result
$results->getResults()[0];

// Get the DTO 🎉 (new!)
$results->getResults()[0]->getModel();

// Create a new version of the Index "beers"
$index = $indexBuilder->createIndex('beers');

// Slow down the Refresh Interval of the new Index to speed up indexation
$indexBuilder->slowDownRefresh($index);
$indexBuilder->speedUpRefresh($index);

// Set proper aliases
$indexBuilder->markAsLive($index, 'beers');

// Clean the old indices (close the previous one and delete the older)
$indexBuilder->purgeOldIndices('beers');
```

*mappings/beers_mapping.yaml*

```yaml
# Anything you want, no validation
settings:
    number_of_replicas: 1
    number_of_shards: 1
    refresh_interval: 60s
mappings:
    dynamic: false
    properties:
        foo:
            type: text
            analyzer: english
            fields:
                keyword:
                    type: keyword
```

## Configuration

This library add custom configurations on top of Elastica's:

### Client::CONFIG_MAPPINGS_DIRECTORY (required)

The directory Elastically is going to look for YAML.

When creating a `foobar` index, a `foobar.yaml` file is expected.

If an `analyzers.yaml` file is present, **all** the indices will get it.

### Client::CONFIG_INDEX_CLASS_MAPPING (required)
 
An array of index name to class FQN.

```php
[
  'indexName' => '\My\AwesomeDTO'
]
```

### Client::CONFIG_SERIALIZER (optional)

A `SerializerInterface` and `DenormalizerInterface` compatible object that will by used both on indexation and search.

_Default to Symfony Object Normalizer._

A faster alternative is to use Jane to generate plain PHP Normalizer, see below. Also we recommend [customization to handle things like Date](https://symfony.com/doc/current/components/serializer.html#normalizers).

### Client::CONFIG_SERIALIZER_CONTEXT_PER_CLASS (optional)

Allow to specify the Serializer context for normalization and denormalization.

```php
$client->setConfigValue(Client::CONFIG_SERIALIZER_CONTEXT_PER_CLASS, [
    Beer::class => ['attributes' => ['title']],
]);
```

_Default to `[]`._

### Client::CONFIG_BULK_SIZE (optional)
    
When running indexation of lots of documents, this setting allow you to fine-tune the number of document threshold. 

_Default to 100._

### Client::CONFIG_INDEX_PREFIX (optional)
    
Add a prefix to all indexes and aliases created via Elastically.

_Default to `null`._

## Usage in Symfony

### Client as a service

Just declare the proper service in `services.yaml`:

```yaml
JoliCode\Elastically\Client:
    arguments:
        $config:
            log: '%kernel.debug%'
            host: '%env(ELASTICSEARCH_HOST)%'
            elastically_mappings_directory: '%kernel.root_dir%/Elasticsearch/mappings'
            elastically_index_class_mapping:
                my_index_name: App\Model\MyModel
            elastically_serializer: '@serializer'
            elastically_bulk_size: 100
```

### Using HttpClient as Transport

You can also use the Symfony HttpClient for all Elastica communications:

```yaml
JoliCode\Elastically\Transport\HttpClientTransport: ~

JoliCode\Elastically\Client:
    arguments:
        $config:
            host: '%env(ELASTICSEARCH_HOST)%'
            transport: '@JoliCode\Elastically\Transport\HttpClientTransport'
            ...
```

### Using Messenger for async indexing

Elastically ships with a default Message and Handler for Symfony Messenger.

Register the message in your configuration:

```yaml
framework:
    messenger:
        transports:
            async: "%env(MESSENGER_TRANSPORT_DSN)%"

        routing:
            # async is whatever name you gave your transport above
            'JoliCode\Elastically\Messenger\IndexationRequest':  async

services:
    JoliCode\Elastically\Messenger\IndexationRequestHandler: ~
```

The `IndexationRequestHandler` service depends on an implementation of `JoliCode\Elastically\Messenger\DocumentExchangerInterface`, which isn't provided by this library. You must provide a service that implements this interface, so you can plug your database or any other source of truth.

Then from your code you have to call:

```php
use JoliCode\Elastically\Messenger\IndexationRequest;
use JoliCode\Elastically\Messenger\IndexationRequestHandler;

$bus->dispatch(new IndexationRequest(Product::class, '1234567890'));

// Third argument is the operation, so for a delete:
// new IndexationRequest(Product::class, 'ref9999', IndexationRequestHandler::OP_DELETE);
```

And then consume the messages:

```sh
$ php bin/console messenger:consume async
```

### Grouping IndexationRequest in a spool

Sending multiple `IndexationRequest` during the same Symfony Request is not always appropriate, it will trigger multiple Bulk operations. Elastically provides a Kernel listener to group all the `IndexationRequest` in a single `MultipleIndexationRequest` message.

To use this mechanism, we send the `IndexationRequest` in a memory transport to be consumed and grouped in a really async transport:

```yaml
messenger:
    transports:
        async: "%env(MESSENGER_TRANSPORT_DSN)%"
        queuing: 'in-memory:///'

    routing:
        'JoliCode\Elastically\Messenger\MultipleIndexationRequest': async
        'JoliCode\Elastically\Messenger\IndexationRequest': queuing
```

You also need to register the subscriber:

```yaml
services:
    JoliCode\Elastically\Messenger\IndexationRequestSpoolSubscriber:
        arguments:
            - '@messenger.transport.queuing' # should be the name of the memory transport
            - '@messenger.default_bus'
        tags:
            - { name: kernel.event_subscriber }
```

## Using Jane to build PHP DTO and fast Normalizers

Install [JanePHP](https://jane.readthedocs.io/) json-schema tools to build your own DTO and Normalizers. All you have to do is setting the Jane-completed Serializer on the Client:

```php
$client = new Client([
    Client::CONFIG_SERIALIZER => $serializer,
]);
```

_[Not compatible with Jane < 6](https://github.com/jolicode/elastically/issues/12)._

## To be done

- some "todo" in the code
- optional Doctrine connector
- better logger - maybe via a processor? extending _log is supposed to be deprecated :(
- optional Symfony integration (DIC)
  - web debug toolbar!
- scripts / commands for common tasks:
  - auto-reindex when the mapping change, handle the aliases and everything
  - micro monitoring for cluster / indexes
  - health-check method

## Sponsors

[![JoliCode](https://jolicode.com/images/logo.svg)](https://jolicode.com)

Open Source time sponsored by JoliCode.
