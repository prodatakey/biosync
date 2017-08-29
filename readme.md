# PDK to BioConnect Sync

This tool syncronizes data from the PDK API to the BioConnect API.

This syncronization facilitates the BioConnect UI in presenting the end user with selection lists filled with data found in the associated PDK system.

When the end user is enrolling a biometric identity they want to enter the user in the pdk system and grant access control rights, then go to the bioconnect system to do the fingerprint enrollment.
They want to be able to see a list of users that are in the pdk system to easily select the enrolee without any double data entry.
They do not want to wait some alloted timeframe for the sync so it needs to be realtime.

At start up a full sync is done to bring the two data sets into alignment.
Once the full sync is complete, data change messages from the PDK API are used to incrementally apply changes to the BioConnect API in realtime.

## Sync Data

A small amount of data needs to be kept in sync:

- First name
- Last name
- Cards
- PIN

Cards are stored as individual entities in the bcAPI like in the pdkAPI, but the PIN is stored on the person in pdk and on the card in bc.
We will have to store the same PIN from the pdk person into each card entity in bc.

We can leverage the bcAPI User entity's `externalId` for storing the pdk person ID.

? How will bioconnect handle facility code of zero? We only store a set of valid facility codes per-system, so we don't know what they are per-card.

## Logic

This is the basic psuedocode for the sync service implementation:

	Create pdk URI to content and ETag mapping cache and tap in to HTTP client to provide content from cache when any new request with ETag returns 304 NOT MODIFIED

	At Startup and any time connection to the pdk message bus is restored:

	Connect to the pdk store message bus

	Get a list of all people in bioconnect store
	Request each person from the pdk store with the cache tag for performance
		If the request response is NOT FOUND
			Delete the person in the bioconnect store
		If the request response is FOUND
			Update the person in the bioconnect store with data received from the pdk store

	Get a list of all people from pdk store
	Compare this list to the the list from the bioconnect store
		If a person is missing from the bioconnect store
			Get person data from the pdk store
				Add the person to the bioconnect store


	Any time there is a person data change message from the pdk store:

	Get person data from pdk store
		Request person from bioconnect store
			If response is NOT FOUND
				Add person to bioconnect store
			If response is FOUND
				Update person in bioconnect store

## Implementation

### CLI

The command line interface will run the service main loop as normal with no parameters and log to stdout.

With the `auth` parameter the interface will execute the pdk API offline authentication flow and persist the `refresh_token` to config.

The interface should check for a bc API and pdk API `client_id` and `client_secret` and a pdk API `refresh_token` then and if any are missing it should print an error to stderr and exit.

### PDK API

We are using the [node-pdk library](https://github.com/prodatakey/node-pdk) to handle authentication and communication with the pdk API.

### BioConnect API

We will be using the [got library](https://github.com/sindresorhus/got) to interact with the bc JSON API endpoints.

### Configuration

The `config.json` file will be used to store configuration settings for the service. Persistent authentication tokens will also be stored here.
