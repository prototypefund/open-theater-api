# Open-Theater-API

API based on socket.io to send & trigger live subtitles/translations for theater and live performance applications to mobile devices in the audience to bridge language barriers.

Translations can be in form of text, video or audio snippets.

<a href="https://www.bmbf.de/"><img src="https://prototypefund.de/wp-content/uploads/2016/07/logo-bmbf.svg"></a>

via:
<a href="https://www.prototypefund.de/">prototypefund.de</a>

## Basic Structure

The API proposals are split into 2 main sections:

1) <a href="#provisioning-api">*Provisioning API*</a> - defining how a repository server, provsioning server and clients communicate and how clients and servers reference cached media Assets in their filesystems
2) <a href="#trigger-api">*Trigger API*</a> - defining how a trigger server forwards triggers/cues to clients to display/play text, video and/or video on client devices.

In the future there might be additional APIs added. In discussion are (amongst others): 
- Renderer Plugin API
- optional Authentication/Ticketing API)

## Status of this document
This document currently is unfinished and not yet in a usable but in an experimental state.

<a href="https://tools.ietf.org/html/rfc2119">RF 2119</a> *is planned* to be used in the first published version of this document.



## Provisioning API

### Terminology

#### Repo List
The Repo List is a optional List of <a href="#repository-definition">repository definitions</a>. It MAY be used to give clients a list of <a href="#repository">repositories</a> to choose from automatically or via user interaction.

If used, the order of repository definitions in the repository list MUST be in the order of priority of the given repository endpoints. In case of an automatic choice by the client, the client MUST try to connect to the repositories in the order they are listed in.

This way it can for example be ensured that clients first try local wifi networks before they switch to wide area networks.

Example:
```json
  { "ssid": "private.open.theater", 
    "pw": "testpassword1234",
    "serveruri": "http://192.168.178.38:8080/mockserver/example-repo/projectList.json"
  }
  {
    "serveruri": "http://192.168.178.38:8080/mockserver/example-repo/projectList.json"
  }
  {
    "serveruri": "https://www.open-theater.de/example-repo/projectList.json"
  }
]
```

#### Repository Definition 

Defines a <a href="#repository">repository</a> endpoint within a <a href="#repo-list">RepoList</a>.

MUST provide parameter `serveruri` containing a String with a valid <a href="https://tools.ietf.org/html/rfc3986">URI</a>

MAY provive parameter `ssid` containing a String defining a wifi ssid if the serveruri MUST be connected in a local area network only

MAY provide parameter `password` containing a String IF parameter `ssid` is present to give access to password protected local wireless networks

The client MUST use the SSID and PASSWORD if provided and MUST NOT use the SERVERURI via a different uplink.
If the client should connect to the same SERVERURI via another uplink in case the wifi connection fails, the repo list MUST provide another repository endpoint definition including the SERVERURI without SSID

Example:
```json
{ "ssid": "private.open.theater", 
"pw": "testpassword1234",
"serveruri": "http://192.168.178.38:8080/mockserver/example-repo/projectList.json"
}

```

#### Repository
A repository (or repo) is a data store that hosts a collection of available <a href="#project">projects</a> and exposes those via a <a href="#project-list">project list</a>

Repositories MAY pull and compile data from other repositories, IF the owners of those repositories allow it. **(still to be defined how)**.

#### Project List
The project list is a text file exposed by a <a href="#repository">repository</a> to clients, listing all <a href="#project">projects</a> the repository provides and/or manages.
It MUST be exposed as `./projectList.json` on a repository's webserver base URL.

#### Project
A Project describes and references a real life event (for example a theater show) that provides <a href="#channel">channels</a> to different media translations.

MUST provide parameter `projectPath` which is a list of path compatible machine readable strings (no spaces, no special characters, case-insensitive) **(todo: needs reference to standard or define its own clear norm, e.g. URI safe path notation)**, describing the path of the project as it SHOULD be displayed by the client to users. Clients MAY hide or show certain layers of the projectPath depending on filters, UX philosophy etc. `projectPath` also describes the path that projects assets downloaded via the provisioning API MUST use within the clients filesystem. Clients MAY prepend to the path to add client specific root directories.

MUST provide parameter `channelList` which is a list of channel objects.

MAY provide parameter `projectLabel` containting a list of human readable strings which MUST have the same length as `projectPath`. The strings inside projectLabel can contain UTF-8 compatible characters that are not allowed inside of `projectPath`. Clients SHOULD map the items from `projectLabel` to items of `projectPath` and preferably display the `projectLabel` Strings to users.
(this way we can display directory structures with spaces and emojis etc if needed).

#### Channel
A channel describes a single media translation of its given parent <a href="#project">project</a>.

MUST provide parameter `channelId` containting a String that is unique within the namespace of its parent <a href="#project">project</a>. clients MUST use the channelId as last directory in the filepath defined by the parents project `projectPath` to cache/save asset media files when provisioning. client MAY use the channelId as human readable identifier IF `label` is not present.

MUST provide parameter `triggerUri` containing a String with a valid <a href="https://tools.ietf.org/html/rfc3986">URI</a> to an Open Theater <a href="#trigger-api-endpoint">trigger API endpoint</a>.

MAY provide parameter `provisioningUri` containing a String with a valid <a href="https://tools.ietf.org/html/rfc3986">URI</a> to an Open Theater <a href="#provisioning-endpoint">provisioning endpoint</a> if files need to be cached before entering the <a href="#trigger-api">trigger API</a>.

MAY provide parameter `channelTypes` containing an Array of strings describing predefined channel content types as `text`,  `video`, `audio` **(to be defined)**. This parameter is to be used for clientside styling and/or UX purposes as well as for automatic detection clientside to which renderer or renderer configuration might be used.

MAY provide parameter `label`containting a String describing the content of the channel in human readable form. Clients SHOULD display the label for users if provided and prefer it over channelId. **(todo: needs character limit recommendations)**

MAY provide parameter `renderer` containing a String identifying the needed renderer for the client IF no standard renderer can be used to display this channels content. IF so this channel's `provisioningUri` endpoints MUST list and deliver the necessary renderer. Clients MAY forbid to download renderers and ignore channels that require custom `renderers`. `renderer` defaults clientside to a defaultRenderer which can display all default `channelTypes` (for more about channelTypes and renderers: see <a href="#trigger-api">Trigger API</a>)

MAY provide a parameter `lastmodified` containing an integer timestamp **(todo: reference standard)** marking the last time any asset inside of the `provisioningUri`s <a href="#fileList">fileList</a> was modified. `lastmodified` MUST only be present IF a `provisioningUri` is provided. IF `lastmodified`is provided a repository's server MUST keep it up to date. RECOMMENDED update timeframe is every 15 minutes. 
Clients MAY use channel.lastmodified to compare with their cached <a href="#fileList">fileList</a> for human readablility and UX (e.g. to signal that a channels ressources have been downloaded before and seem up to date) but they MUST NOT use it automatically to skip any checks with the provisioning endpoint of a given channel before entering the trigger API.

<!-- CONTINUE HERE -->
<img src="https://open-theater.de/wp-content/uploads/2020/12/OpenTheater-API-Flow-und-Visual-Doku-1.jpg" alt="flowchart of provisioning uri with comments"></img>
