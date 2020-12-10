# Open-Theater-API

API based on socket.io to send & trigger live subtitles/translations for theater and live performance applications to mobile devices in the audience to bridge language barriers.

Translations can be in form of text, video or audio snippets.

<a href="https://www.bmbf.de/"><img src="https://prototypefund.de/wp-content/uploads/2016/07/logo-bmbf.svg"></a>

via:
<a href="https://www.prototypefund.de/">prototypefund.de</a>

## Basic Structure

The API proposals are split into 2 main sections:

1) *Provisioning API* - defining how a repository server, provsioning server and clients communicate and how clients and servers reference cached media Assets in their filesystems
2) *Trigger API* - defining how a trigger server forwards triggers/cues to clients to display/play text, video and/or video on client devices.

In the future there might be additional APIs added. In discussion are (amongst others): 
- Renderer Plugin API
- optional Authentication/Ticketing API)

## Status of this document
This document currently is unfinished and not yet in a usable but in an experimental state.

<a href="https://tools.ietf.org/html/rfc2119">RF 2119</a> *is planned* to be used in the first published version of this document.



## Provisioning API

### Terminology

#### Repo List
The Repo List is a optional List of repository definitions. It MAY be used to give clients a list of repositories to choose from automatically or via user interaction.

If used, the order of repository endpoint definitions in the repository list MUST be in the order of priority of the given repository endpoints. In case of an automatic choice by the client, the client MUST try to connect to the repositories in the order they are listed in.

This way it can for example be ensured that clients first try local wifi networks before they switch to wide area networks.

Example:
```json
  { "ssid": "private.open.theater", 
    "pw": "testpassword1234",
    "serveruri": "http://192.168.178.38:8080/mockserver/example-repo/services.json"
  }
  {
    "serveruri": "http://192.168.178.38:8080/mockserver/example-repo/services.json"
  }
  {
    "serveruri": "https://www.open-theater.de/example-repo/services.json"
  }
]
```

#### Repository Endpoint Defintion 

Defines a repository endpoint within a RepoList.

MUST provide parameter `serveruri` containing a String with a valid <a href="https://tools.ietf.org/html/rfc3986">URI</a>

MAY provive parameter `ssid` containing a String defining a wifi ssid if the serveruri MUST be connected in a local area network only

MAY provide parameter `password` containing a String IF parameter `ssid` is present to give access to password protected local wireless networks

The client MUST use the SSID and PASSWORD if provided and MUST NOT use the SERVERURI via a different uplink.
If the client should connect to the same SERVERURI via another uplink in case the wifi connection fails, the repo list MUST provide another repository endpoint definition including the SERVERURI without SSID

Example:
```json
{ "ssid": "private.open.theater", 
"pw": "testpassword1234",
"serveruri": "http://192.168.178.38:8080/mockserver/example-repo/services.json"
}

```

#### Repository
A repository (or repo) is a data store that hosts a collection of available <a href="#Project">projects</a> and exposes those via a <a href="#Project List">project list</a>

Repositories MAY pull and compile data from other repositories, IF the owners of those repositories allow it. **(still to be defined how)**.

#### Project List
The project list is a text file exposed by a repository to clients, listing all projects the reposiotory provides and manages.
It MUST be exposed as `./projectList.json` on a repository's webservers root URL.

#### Project
A Project describes and references a real life event (for example a theater show) that provides channels to different media translations.

MUST provide parameter `projectPath` which is a list of Strings, describing the path of the project as it SHOULD be displayed by the client to users. Clients MAY hide or show certain layers of the projectPath depending on filters, UX philosophy etc. `projectPath` also describes the path that projects assets downloaded via the provisioning API MUST use within the clients filesystem. Clients MAY prepend to the path to add client specific root directories.

MUST provide parameter `channelList` which is a list of channel objects.

#### Channel
A channel describes a single media translation of its given parent project.

MUST provide parameter `triggerUri` containing a String with a valid <a href="https://tools.ietf.org/html/rfc3986">URI</a> do a Open Theater triggerAPI endpoint

MAY provice parameter `channelType` containing a String describing predefined channel content types as `text`,  `video`, `audio` **(to be defined)**. This parameter is to be used for clientside styling and/or UX purposes.

<!-- CONTINUE HERE -->

