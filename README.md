[[_TOC_]]

# Open-Theater-API

API based on socket.io and HTTP to send, provision & trigger live subtitles/translations for theater and live performance applications to mobile devices in the audience to bridge language barriers.

Translations can be in form of text, video or audio snippets.

The demo implementation for a mobile client and mockserver can be found at https://gitlab.com/open-theater/open-theater-client-capacitorjs

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


## Agents of API processes

#### Client
A client. Usually an application running on one single user's device (mobile phone, laptop etc.).

#### Repository Server
A server hosting an overview over several projects that MAY recide on different servers (provisioning servers and/or trigger servers).

#### Provisioning Server
A server hosting one or more project and channel specific provisioning endpoints and/or their fileLists and/or their associated asset files to be downloaded and chached by clients in the provisioning process.

#### Trigger Server
A server hosting one or more project and/or channel specific trigger endpoints. Delivers real time trigger payloads to clients that are subscribed to its endpoints.

## Provisioning API

status: test ready

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

MUST provide field `serveruri` containing a String with a valid <a href="https://tools.ietf.org/html/rfc3986">URI</a>

MAY provive field `ssid` containing a String defining a wifi ssid if the serveruri MUST be connected in a local area network only

MAY provide field `password` containing a String IF field `ssid` is present to give access to password protected local wireless networks

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

MUST provide field `projectUuid` which is a string formatted as <a href="https://tools.ietf.org/html/rfc4122#section-4.1.3">UUID V4</a>.

MUST provide field `projectPath` which is a list of path compatible machine readable strings (no spaces, no special characters, case-insensitive) **(todo: needs reference to standard or define its own clear norm, e.g. URI safe path notation)**, describing the path of the project as it SHOULD be displayed by the client to users. Clients MAY hide or show certain layers of the projectPath depending on filters, UX philosophy etc. `projectPath` also describes the path that projects assets downloaded via the provisioning API MUST use within the clients filesystem. Clients MAY prepend to the path to add client specific root directories.

MUST provide field `channelList` which is a list of channel objects.

MUST provide field `triggerUri` containing a string with a valid <a href="https://tools.ietf.org/html/rfc3986">URI</a> to an Open Theater <a href="#trigger-api-endpoint">trigger API endpoint</a>.

MAY provide field `projectLabel` containting a list of human readable strings which MUST have the same length as `projectPath`. The strings inside projectLabel can contain UTF-8 compatible characters that are not allowed inside of `projectPath`. Clients SHOULD map the items from `projectLabel` to items of `projectPath` and preferably display the `projectLabel` Strings to users.
(this way we can display directory structures with spaces and emojis etc if needed).

Example:
```json
{
    "projectUuid": "ff277516-121b-4304-b256-d5a8b70e136a",
    "projectPath": [
      "Staatstheater_Mannheim",
      "Kammerspiele",
      "Shakespears_anderes_stück"
    ],
    "triggerUri": "https://foo.bar/project2",
    "channelList": [
      {
        "channelUuid": "f6b070fe-0f89-4c1e-8c43-993abd9e851f",
        "provisioningUri": "https://open-theater.de/example-repo/staatstheater_mannheim/kammerspiele/shakespears_anderes_stueck/DE_UNTERTITEL/fileList.json",
        "label": "DE Untertitel",
        "containerIds": ["text_DE"]
      },
      {
        "channelUuid": "5e23407e-1d35-4953-bf17-5e07e11105d4",
        "provisioningUri": "https://open-theater.de/example-repo/staatstheater_mannheim/kammerspiele/shakespears_anderes_stueck/EN_UNTERTITEL/fileList.json",
        "label": "EN Subtitles",
        "containerIds": ["text_EN"]
      }
    ]
}

```

#### Channel
A channel describes a single media translation of its given parent <a href="#project">project</a>.

MUST provide field `channelUuid` containting a string formatted as  <a href="https://tools.ietf.org/html/rfc4122#section-4.1.3">UUID V4</a>. clients MUST use the channelUuid as last directory in the filepath defined by the parents project `projectPath` to cache/save asset media files when provisioning. client MAY use the channelUuid as human readable identifier IF `label` is not present.

MAY provide field `triggerUri` containing a String with a valid <a href="https://tools.ietf.org/html/rfc3986">URI</a> to an Open Theater <a href="#trigger-api-endpoint">trigger API endpoint</a> that CAN overwrite the parent project level `triggerUri` (for example to have slimmer data packets per cue). This CAN be used by a client but a client MAY also choose to default to the more verbose project's `triggerUri`. We RECOMMEND using this only in projects with excessive data being send over a high number of channels and otherwise omit this `triggerUri`.

MAY provide field `provisioningUri` containing a String with a valid <a href="https://tools.ietf.org/html/rfc3986">URI</a> to an Open Theater <a href="#provisioning-endpoint">provisioning endpoint</a> if files need to be cached before entering the <a href="#trigger-api">trigger API</a>.

<!--MUST provide field `channelTypes` containing an Array of strings describing predefined channel content types as `text`,  `video`, `audio` **(to be defined)**. This field is to be used for clientside styling and/or UX purposes as well as for automatic detection clientside to which renderer or renderer configuration might be used.-->

MUST provide field `containerIds` containting an Array of strings, each describing the unique id of each content div (track) that the <a href="#renderer">renderer</a> of this channel will have to expect in incoming <a href="#trigger-payload">trigger payloads (cues)</a>.
Each key MUST be formatted following the format `renderingMediumType_languagelabel`, where `renderingMediumType` should be replaced by the predefined channel content types  `text`,  `video`, `audio`, `image` **(to be defined)** and `languagelabel` should be replaced by the <a href="https://en.wikipedia.org/wiki/ISO_639-1">ISO 639-1 language code</a> (if applicable) or another custom flag identifying this channel. This parameter is to be used for clientside styling and/or UX purposes as well as for automatic detection clientside to which renderer or renderer configuration might be used. It will be used as ID of each div built by the renderer and as the keys identifying distinct  in the trigger API.
<!--
**Spec Note**: *this spec might be worth re-examination for later versions of this spec for it seems a little too unintuitive. Yet at the current moment it seems necessary to have flexible renderers*

Might be replaceable automatically by the more human readable channelTypes_channelLabel ! TODO: check if this would break some logic in the renderer and/or other parts of the spec!
-->

MAY provide field `label`containting a String describing the content of the channel in human readable form. Clients SHOULD display the label for users if provided and prefer it over channelUuid. **(todo: needs character limit recommendations)**

MAY provide field `renderer` containing a String identifying the needed renderer for the client IF no standard renderer can be used to display this channels content. IF so this channel's `provisioningUri` endpoints MUST list and deliver the necessary renderer. Clients MAY forbid to download renderers and ignore channels that require custom `renderers`. `renderer` defaults clientside to a defaultRenderer which can display all default `channelIds` (for more about channelIds and renderers: see <a href="#trigger-api">Trigger API</a>)

MAY provide a field `lastmodified` containing an integer timestamp **(todo: reference standard)** marking the last time any asset inside of the `provisioningUri`s <a href="#fileList">fileList</a> was modified. `lastmodified` MUST only be present IF a `provisioningUri` is provided. IF `lastmodified`is provided a repository's server MUST keep it up to date. RECOMMENDED update timeframe is every 15 minutes. 
Clients MAY use channel.lastmodified to compare with their cached <a href="#fileList">fileList</a> for human readablility and UX (e.g. to signal that a channels ressources have been downloaded before and seem up to date) but they MUST NOT use it automatically to skip any checks with the provisioning endpoint of a given channel before entering the trigger API.

#### File List
A JSON file hosted on a provisioning server and within a client's filesystem for each channel cached.

MUST be called `fileList.json`.

MUST be served at root level relative to it's <a href="#channel">channel's</a> `provisioningUri` IF on provisioning servers.

MUST be saved at root level relative to it's channel's cached file location IF on client filesystem.

MUST provide field `files`, containing a list of objects with <a href="#file-definition">File Definitions</a>.

Example:
```json
{
    "files":[
        { "filepath": "1.mp4", "filesize": 12, "lastmodified": 16037178357021 },
        { "filepath": "2.mp4", "filesize": 64, "lastmodified": 16037178356002 },
        { "filepath": "3.mp4", "filesize": 64, "lastmodified": 16037178356002 },
        { "filepath": "4.mp4", "filesize": 64, "lastmodified": 16037178357004 }
    ]
}

```

#### File Definition
Part of <a href="#file-list">file list</a>. Describes one asset file for caching in the provisioning API.

MUST provide field `filepath` containing a URI path compatible string, describing EITHER the relative position of an asset file to the fileList's own location (defined by the `provisioningUri` of it's parent channel) OR an absolute position defined by the URI.

MUST provide field `lastmodified` containting an integer value describing a timestamp in <a href="https://en.wikipedia.org/wiki/Unix_time">Unix Time</a>.

SHOULD provide field `filesize`, containting an integer value describing the file's size in bytes.

Provisioning servers MUST make sure that the listed filepaths are accessible by clients when listed. We RECOMMEND to test regularly and automated all listed filepaths.

Example:
```json
{ "filepath": "1.mp4", "filesize": 12, "lastmodified": 16037178357021 }

```

<!-- CONTINUE HERE -->

### Data flow of provisioning process

Clients MUST follow the following steps in order:

1. On boot: Client opens <a href="#repo-list">Repo List</a> and works through the <a href="#repository-definition">Repository Definitions</a> in order of their listing following the rules of the specifications. (IF ssid/pw present: use local wifi, else use WAN etc.). Client uses the response data from the first successful connection to a listed repositoriy server <a href="#project-list">project list</a>.

2. Client presents all relevant projects and their channels from the project list to users. Client MAY apply filters based on their capabilities (e.g. custom renderers) and configurations (e.g. user choices). Client CAN choose the styling and order it presents the contents to the user, but MUST follow the specifications of <a href="#project">project</a> and <a href="#channel">channel</a> (`label`over `channelUuid`, etc. etc.).

3. Client / User chooses one or more channels from the project list.

4. Client connects via HTTP GET request to the provisioning endpoint's `/fileList.json` of each chosen channel

5. Client uses the <a href="#file-list">file List</a> to compare ALL files listed on the channels provisioning endpoint with ALL files cached inside the client's filesystem by using the locally cached fileList

6. Client downloads ALL files that are not up to date with the files chached and updates it's channel specific local fileList accordingly.

7. Client waits until ALL updates and downloads of AT LEAST one channel are finished

8. Client provides user possiblity to access a project's trigger API and it's user interface. (e.g. via button press). Client MUST have ensured that Step 7 has been finished for the project before letting users access the trigger client.

9. Client hands over all information about the project chosen to the trigger client.


<!-- TODO: add more details over decision trees in the flow here. -->

<img src="https://open-theater.de/wp-content/uploads/2020/12/OpenTheater-API-Flow-und-Visual-Doku-1.jpg" alt="flowchart of provisioning uri with comments"></img>

### Provisioning API Notes

- channels MAY have their own Trigger endpoints, only listing content and style changes for one channel, but every project MUST have a verbose trigger endpoint including ALL trigger payloads for ALL channels

## Trigger API

status: pre-alpha

### Terminology

#### Show Interface

A User Interface provided by a client to display and/or play contents described in <a href="#trigger-payload">a trigger playload</a>. 

MUST include at least one <a href="#renderer">renderer</a> to present contents to users. At least one renderer MUST serve as the <a href="#default-renderer>default renderer</a>.

MAY switch to <a href="#custom-renderer">custom renderers</a> IF trigger payload, <a href="project">project</a> and/or client configuration allow.

#### Trigger Payload

A JSON formatted payload describing the contents to be displayed and/or played within a client's <a>show interface's</a> <a href="#renderer">renderers</a>.

MUST provide at least one of the following fields `content` and/or `styles`. MAY only describe one of them as changes.

MAY provide field `content` containing a <a href="#content-object">content-object</a> of <a href="#content">contents</a> of each <a href="#channel">channel</a>, describing the unstyled changes of content on receiving of this trigger payload.

MAY provide field `styles` containing a <a href="#styles-object">styles-object</a>, describing the styling changes valid on receiving this trigger payload.

SHOULD provide field `timestamp` containting an integer value describing a timestamp in <a href="https://en.wikipedia.org/wiki/Unix_time">Unix Time</a> indicating the time this payload was sent by a <a href="#cue-generator">cue-generator</a>

Example:
```json
{
  "timestamp":123445667778,
  "content":{

    "text_de":"das ist deutsch  ${count}",

    "text_en":"this is english  ${count}",

    "html_doesnotexist": "test",

    "video_de": "assets/test.mp4",

    "image_de": "https://picsum.photos/300/300"

  },
  "styles": {
      "#text_de": {
        "classes": [],
        "inline": {
          "backgroundColor": "yellow",
          "border": "4px solid red"
        },
        "transitions": {
          "fadeInClass": "animate__fadeIn",
          "fadeInTime": 100,
          "fadeInDelay": 100,
          "fadeOutClass": "animate__fadeOut",
          "fadeOutTime": 100,
          "fadeOutDelay": 100
        }
    }
  }
}
```

#### Content Object

A content Object is part of a <a href="#trigger-payload">trigger payload</a> and describes the content changes of one trigger cue. 
It lists the contents for all channels that MUST change immediately after receiving the parent trigger payload. 

MUST list the changes as key-value pairs, where the key MUST be lead by the name of the renderer the change is affecting and an underscore, followed by  a valid channel name as defined in the provisioning's <a href="#channel">channel definitions</a>. For example `text_german` for the text renderer on channel "german" being affected.
The value can be whatever data structure is required by the renderer. For all default renderers that will be a string.

Example:
```
"content":{

    "text_de":"das ist deutsch  ${count}",

    "text_en":"this is english  ${count}",

    "html_doesnotexist": "test",

    "video_de": "assets/test.mp4",

    "image_de": "https://picsum.photos/300/300"

  },
```

#### Styles Object



### Trigger API Notes / Flow

- Because the <a href="#trigger-payload">Trigger Payload</a> describes changes to single channels, it MUST be delivered via TCP based transport protocols or other means to ensure delivery on transport level. If UDP based / one-way transport protocols were used, every trigger payload would have to repeat the state instead of only overwriting. For later versions of this API specification there could be a flag on channels.
