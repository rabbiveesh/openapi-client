# NAME

OpenAPI::Client - A client for talking to an Open API powered server

# DESCRIPTION

[OpenAPI::Client](https://metacpan.org/pod/OpenAPI%3A%3AClient) can generating classes that can talk to an Open API server.
This is done by generating a custom class, based on a Open API specification,
with methods that transform parameters into a HTTP request.

The generated class will perform input validation, so invalid data won't be
sent to the server.

Note that this implementation is currently EXPERIMENTAL, but unlikely to change!
Feedback is appreciated.

# SYNOPSIS

## Open API specification

The specification given to ["new"](#new) need to point to a valid OpenAPI document.
This document can be OpenAPI v2.x or v3.x, and it can be in either JSON or YAML
format. Example:

    openapi: 3.0.1
    info:
      title: Swagger Petstore
      version: 1.0.0
    servers:
    - url: http://petstore.swagger.io/v1
    paths:
      /pets:
        get:
          operationId: listPets
          ...

`host`, `basePath` and the first item in `schemes` will be used to construct
["base\_url"](#base_url). This can be altered at any time, if you need to send data to a
custom endpoint.

## Client

The OpenAPI API specification will be used to generate a sub-class of
[OpenAPI::Client](https://metacpan.org/pod/OpenAPI%3A%3AClient) where the "operationId", inside of each path definition, is
used to generate methods:

    use OpenAPI::Client;
    $client = OpenAPI::Client->new("file:///path/to/api.json");

    # Blocking
    $tx = $client->listPets;

    # Non-blocking
    $client = $client->listPets(sub { my ($client, $tx) = @_; });

    # Promises
    $promise = $client->listPets_p->then(sub { my $tx = shift });

    # With parameters
    $tx = $client->listPets({limit => 10});

See [Mojo::Transaction](https://metacpan.org/pod/Mojo%3A%3ATransaction) for more information about what you can do with the
`$tx` object, but you often just want something like this:

    # Check for errors
    die $tx->error->{message} if $tx->error;

    # Extract data from the JSON responses
    say $tx->res->json->{pets}[0]{name};

Check out ["error" in Mojo::Transaction](https://metacpan.org/pod/Mojo%3A%3ATransaction#error), ["req" in Mojo::Transaction](https://metacpan.org/pod/Mojo%3A%3ATransaction#req) and
["res" in Mojo::Transaction](https://metacpan.org/pod/Mojo%3A%3ATransaction#res) for some of the most used methods in that class.

# CUSTOMIZATION

## Custom server URL

If you want to request a different server than what is specified in the Open
API document, you can change the ["base\_url"](#base_url):

    # Pass on a Mojo::URL object to the constructor
    $base_url = Mojo::URL->new("http://example.com");
    $client1 = OpenAPI::Client->new("file:///path/to/api.json", base_url => $base_url);

    # A plain string will be converted to a Mojo::URL object
    $client2 = OpenAPI::Client->new("file:///path/to/api.json", base_url => "http://example.com");

    # Change the base_url after the client has been created
    $client3 = OpenAPI::Client->new("file:///path/to/api.json");
    $client3->base_url->host("other.example.com");

## Custom content

You can send XML or any format you like, but this require you to add a new
"generator":

    use Your::XML::Library "to_xml";
    $client->ua->transactor->add_generator(xml => sub {
      my ($t, $tx, $data) = @_;
      $tx->req->body(to_xml $data);
      return $tx;
    });

    $client->addHero({}, xml => {name => "Supergirl"});

See [Mojo::UserAgent::Transactor](https://metacpan.org/pod/Mojo%3A%3AUserAgent%3A%3ATransactor) for more details.

# EVENTS

## after\_build\_tx

    $client->on(after_build_tx => sub { my ($client, $tx) = @_ })

This event is emitted after a [Mojo::UserAgent::Transactor](https://metacpan.org/pod/Mojo%3A%3AUserAgent%3A%3ATransactor) object has been
built, just before it is passed on to the ["ua"](#ua). Note that all validation has
already been run, so alternating the `$tx` too much, might cause an invalid
request on the server side.

A special ["env" in Mojo::Message::Request](https://metacpan.org/pod/Mojo%3A%3AMessage%3A%3ARequest#env) variable will be set, to reference the
operationId:

    $tx->req->env->{operationId};

Note that this usage of `env()` is currently EXPERIMENTAL:

# ATTRIBUTES

## base\_url

    $base_url = $client->base_url;

Returns a [Mojo::URL](https://metacpan.org/pod/Mojo%3A%3AURL) object with the base URL to the API. The default value
comes from `schemes`, `basePath` and `host` in the OpenAPI v2 specification
or from `servers` in the OpenAPI v3 specification.

## ua

    $ua = $client->ua;

Returns a [Mojo::UserAgent](https://metacpan.org/pod/Mojo%3A%3AUserAgent) object which is used to execute requests.

# METHODS

## call

    $tx = $client->call($operationId => \%params, %content);
    $client = $client->call($operationId => \%params, %content, sub { my ($client, $tx) = @_; });

Used to either call an `$operationId` that has an "invalid name", such as
"list pets" instead of "listPets" or to call an `$operationId` that you are
unsure is supported yet. If it is not, an exception will be thrown,
matching text "No such operationId".

`$operationId` is the name of the resource defined in the
[OpenAPI specification](https://github.com/OAI/OpenAPI-Specification/blob/master/versions/2.0.md#operation-object).

`$params` is optional, but must be a hash ref, where the keys should match a
named parameter in the [OpenAPI specification](https://github.com/OAI/OpenAPI-Specification/blob/master/versions/2.0.md#parameter-object).

`%content` is used for the body of the request, where the key need to be
either "body" or a matching ["generators" in Mojo::UserAgent::Transactor](https://metacpan.org/pod/Mojo%3A%3AUserAgent%3A%3ATransactor#generators). Example:

    $client->addHero({}, body => "Some data");
    $client->addHero({}, json => {name => "Supergirl"});

`$tx` is a [Mojo::Transaction](https://metacpan.org/pod/Mojo%3A%3ATransaction) object.

## call\_p

    $promise = $client->call_p($operationId => $params, %content);
    $promise->then(sub { my $tx = shift });

As ["call"](#call) above, but returns a [Mojo::Promise](https://metacpan.org/pod/Mojo%3A%3APromise) object.

## new

    $client = OpenAPI::Client->new($specification, \%attributes);
    $client = OpenAPI::Client->new($specification, %attributes);

Returns an object of a generated class, with methods generated from the Open
API specification located at `$specification`. See ["schema" in JSON::Validator](https://metacpan.org/pod/JSON%3A%3AValidator#schema)
for valid versions of `$specification`.

Note that the class is cached by perl, so loading a new specification from the
same URL will not generate a new class.

Extra `%attributes`:

- app

    Specifying an `app` is useful when running against a local [Mojolicious](https://metacpan.org/pod/Mojolicious)
    instance.

- coerce

    See ["coerce" in JSON::Validator](https://metacpan.org/pod/JSON%3A%3AValidator#coerce). Default to "booleans,numbers,strings".

## validator

    $validator = $client->validator;
    $validator = $class->validator;

Returns a [JSON::Validator::Schema::OpenAPIv2](https://metacpan.org/pod/JSON%3A%3AValidator%3A%3ASchema%3A%3AOpenAPIv2) object for a generated class.
Note that this is a global variable, so changing the object will affect all
instances returned by ["new"](#new).

# COPYRIGHT AND LICENSE

Copyright (C) 2017-2021, Jan Henning Thorsen

This program is free software, you can redistribute it and/or modify it under
the terms of the Artistic License version 2.0.

# AUTHORS

Jan Henning Thorsen - `jhthorsen@cpan.org`

Ed J - `etj@cpan.org`
