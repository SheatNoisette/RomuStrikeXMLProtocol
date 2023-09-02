# RomuStrike XML server protocol

## Introduction
The RomuStrike XML server protocol is a simple protocol for communicating with 
a RomuStrike server. It is based on XML and the game parses it
with the MSXMLv2 library.

This document describes the v153 protocol implemented by RomuStrike v1.7.0.

Please note that this document is not complete and may contain errors. A lot
of the information in this document is based on reverse engineering, network
sniffing and partial source code.

This document is licensed under the Creative Commons
Attribution-NonCommercial-ShareAlike 4.0 International license. You can find
the license [here](https://creativecommons.org/licenses/by-nc-sa/4.0/)

Authors:
- [SheatNoisette](https://github.com/SheatNoisette): Reverse, main documentation
- [Arkhaal](https://github.com/Stalker2106x): Server, conformance testing

## The protocol

The game sends a request to the server and the server responds with a reply
using the GET method. The request is sent as a query string and the reply is
sent as an dumb XML document encoded in ISO-8859-1.

All of requests are sent on the "xml_layer.php" endpoint, more specifically
`http://romustrike.fr/script/romustrike/xml_layer.php`. Only one argument is
required: `crypt`, which is the "encrypted" request.

There's a simple example of a request:
```
GET /script/romustrike/xml_layer.php?crypt=15C6075685E613C6754715E6955234D457D4A46497F38707754747665174944507056523D407264715472274C74564E46653833267D4457456356484F7F38303506 HTTP/1.1
Accept: */*
Accept-Language: en-us
Accept-Encoding: gzip, deflate
User-Agent: Mozilla/4.0 (compatible; MSIE 7.0; Windows NT 6.2; Trident/7.0; .NET4.0C; .NET4.0E; Tablet PC 2.0)
Host: 192.168.1.43:4269
Connection: Keep-Alive
```

In this example, the host is `192.168.1.43:4269` and the `crypt` argument is
`15C6075685E613C6754715E6955234D457D4A46497F38707754747665174944507056523D407264715472274C74564E46653833267D4457456356484F7F38303506`.

## The encryption

The encryption is a simple XOR encryption using a hardcoded 32b key in the game.
The first character of the encrypted ASCII string is not used at all. The
payload is encoded in hexadecimal as a string.

Let's take a simple example, `15C6000`:
```
1: Is Unused
hex(53): Data
hex(43): Data
hex(00): Data
```

The encryption is done using the `CXmlMessaging::Crypt` method. The decryption
is done on the server side, no decryption is done in the game as the response
if a simple XML document.

There's a simple implementation of the decryption in Python:
```python
# This function is under the GPL v3 license
def decrypt(payload, key):
    circular_buffer = [(key >> (8 * i)) % 256 for i in range(4)]

    # Remove useless 1 in the beginning
    payload = payload[1:]

    extracted_payload = []
    for i in range(0, len(payload), 2):
        extracted_payload.append(int(payload[i:i+2], 16))

    decrypted = ""
    for i in range(len(extracted_payload)):
        decrypted += chr(extracted_payload[i] ^ circular_buffer[i % 4])

    return decrypted
```

Getting the key is out of scope. We will not provide the key here, you can find
it by yourself by using brute force or by reverse engineering the game.

## The request

The encrypted request done by the game use the same format as a web request:
```
method=example&example_data=2&LAVERSION=157
```

The first argument is the method, the other arguments are the data. The end of
the request is marked by the `LAVERSION` argument. The `LAVERSION` argument is
the version of the game, it is used by the server to check if the client is
up-to-date. Between the `method` argument and the `LAVERSION` argument, there's
a `&` character which delimits the arguments. The arguments can be easily
parsed by splitting the string with the `&` character then splitting each
argument with the `=` character.

## The response

As mentioned before, the response is a simple XML document. The XML document
has a root element which can be named whatever you `root` or `xml` and
a header which few informations on the XML document
(`<?xml version="1.0" encoding="ISO-8859-1"?>`)
The header doesn't seems to be used by the game but should be present in the
response just in case. The XML document is encoded in ISO-8859-1.

The XML document has a simple structure:
```xml
<?xml version="1.0" encoding="ISO-8859-1"?>
<root><ID_PLAYER>4269</ID_PLAYER></root>
```

The game doesn't check proprely the XML, only basic XML syntax is checked.
The content however is checked at all. If the XML document is syntactically
not valid, the message "erreur accès xml" is shown. Most of the data extracted
from the XML document are extracted using atoi, atof and strcpy. This means
that the game is vulnerable to buffer overflows and integer overflows.

Be careful when you send a response to the game and make sure that the server
you're using is not malicious. One way to prevent this is to use a local server
which is a proxy to the real server using modern encryption.

## The methods

The game has multiples methods which each have a different purpose:
* nouveaujoueur
* get_map
* get_mp3
* delete_server
* quitter_server
* set_tournois
* score_plus
* set_mp3
* set_server
* get_server
* get_id
* info_joueur
* joinserver
* get_tournois
* info_tournois
* set_objet
* get_objet

These methods are used to communicate with the server using the `method`
argument. The `method` argument is the name of the method. The other arguments
are the data.

## Authentication

When you connect to a server, the server sends a `get_id` request to the
RomuStrike server. The `get_id` request is used to get the player ID. If the
player ID is not found, the game shows a error. If the player ID is found, the
game send `info_joueur` request to get the player information. Finally, the
`get_mp3` request is sent to get the list of the server.

## Game methods

To do a request, you need to send a `method` argument. The `method` argument
is the name of the method. The other arguments are the data.

The payload in this context is what is between the `method=%s`
and `LAVERSION=%i`. For example, if the payload `test=%i&test2=%s` is given,
the real payload recieved by the server is
`method=%s&test=%i&test2=%s&LAVERSION=%i`.

The payload are given as a unparsed C string. The `%s` format is used to
replace a string, the `%d` or `%i` format is used to replace a number and `%f`
represent a 32 bits IEEE 754 floating point number.

The acronym `TBD` is used when the payload/response is not fully understood or
not yet documented.

The payload may be different based on the circumstances, in this case, the
given as a regular expression.
For example:
```
test=%i&test2=%s&(testa=%i|testb=%s)&test3=%i
```
This means that the payload can have `testa` or `testb` but not both.

### nouveaujoueur

This method is used to create a new player. It is not used on the most recent
servers.

Payload:
```
LENUM=0&LESOFT=2&LENOM=%s&LEVERSION=100&LEMAIL=%s&LEPASS=%s
```

Description of the arguments:
| Argument  | Type   | Description                                             |
| --------- | ------ |---------------------------------------------------------|
| LENUM     | Number | The player ID, default set to 0  |
| LESOFT    | Number | The software ID, default set to 2 |
| LENOM     | String | The player name |
| LEVERSION | Number | The game version, default set to 100 |
| LEMAIL    | String | The player email |
| LEPASS    | String | The raw player password |

Response:
```xml
<TBD>
```

### get_map

Get the list of the current running maps.

Payload:
```
LENUM=%i&LEPASS=%s&LESOFT=2
```

Description of the arguments:
| Argument  | Type   | Description                                             |
| --------- | ------ |---------------------------------------------------------|
| LENUM     | Number | The player ID |
| LESOFT    | Number | The software ID |
| LEPASS    | String | The raw player password |

Response:
```
<?xml version="1.0" encoding="ISO-8859-1"?>
<root>
    <map>
        <NAME>%s</NAME>
        <MAPPEUR>%s</MAPPEUR>
        <WADMD5>%s</WADMD5>
        <BSPMD5>%s</BSPMD5>
        <HOST>%s</HOST>
    <map>
    <!-- ... -->
</root>
```

Description of the response:
| Argument  | Type   | Description                                             |
| --------- | ------ |---------------------------------------------------------|
| NAME      | String | The map name |
| MAPPEUR   | String | The map author |
| WADMD5    | String | TBD, maybe a MD5 hash of the map WAD |
| BSPMD5    | String | TBD, maybe a MD5 hash of the map BSP |
| HOST      | String | TBD |

### get_mp3

Get the list of the current servers.

Payload:
```
LENUM=%i&LEPASS=%s&LESOFT=2
```

Description of the arguments:
| Argument  | Type   | Description                                             |
| --------- | ------ |---------------------------------------------------------|
| LENUM     | Number | The player ID |
| LESOFT    | Number | The software ID |
| LEPASS    | String | The raw player password |

Response:
```
<?xml version="1.0" encoding="ISO-8859-1"?>
<root>
    <element NAME="%s" COMMENTAIRE="%s" HOST="%s" ID="%i"/>
    <!-- ... -->
</root>
```

Description of the response:
| Argument    | Type   | Description                                           |
| ---------   | ------ |-------------------------------------------------------|
| NAME        | String | TBD |
| COMMENTAIRE | String | TBD |
| HOST        | String | TBD |
| ID          | Number | TBD |

### delete_server

Payload:
```
LENUM=%i&LEPASS=%s&CLE_SERVEUR=%i
```

Description of the arguments:
| Argument    | Type   | Description                                           |
| ---------   | ------ |-------------------------------------------------------|
| LENUM       | Number | The player ID |
| LEPASS      | String | The raw player password |
| CLE_SERVEUR | Number | The internal server ID |

No response is required.

### quitter_server

Exit the server.

Payload:
```
LENUM=%i&LEPASS=%s&LESCORE=%i&LAPARTIE=%i&KILLER=%i&KILLED=%i
```

Description of the arguments:
| Argument  | Type   | Description                                             |
| --------- | ------ |---------------------------------------------------------|
| LENUM     | Number | The player ID |
| LEPASS    | String | The raw player password |
| LESCORE   | Number | TBD |
| LAPARTIE  | Number | TDB |
| KILLER    | Number | TDB |
| KILLED    | Number | TDB |

No response is required.

### set_tournois

Create a new tournament.

Payload:
```
LENUM=%i&LEPASS=%s&CLE_TOURNOIS=%i&ROUND=%i&SCORE1=%i&SCORE2=%i
```

Description of the arguments:
| Argument     | Type   | Description                                           |
| ---------    | ------ |-------------------------------------------------------|
| LENUM        | Number | The player ID |
| LEPASS       | String | The raw player password |
| CLE_TOURNOIS | Number | The internal tournament ID |
| ROUND        | Number | TBD, round number |
| SCORE1       | Number | TBD |
| SCORE2       | Number | TBD |

No response is required.

### score_plus

Ask the server to add a score to the player.

Payload:
```
LENUM=%i&LEPASS=%s&LESCORE=%i&LAPARTIE=%i(&KILLER=%i|&KILLED=%i)
```

Description of the arguments:
| Argument  | Type   | Description                                             |
| --------- | ------ |---------------------------------------------------------|
| LENUM     | Number | The player ID |
| LEPASS    | String | The raw player password |
| LESCORE   | Number | TBD |
| LAPARTIE  | Number | TDB |
| KILLER    | Number | TDB |
| KILLED    | Number | TDB |

No response is required.

### set_mp3

Payload:
```
LENUM=%i&LEPASS=%s&IDMP3=%i
```

Description of the arguments:
| Argument  | Type   | Description                                             |
| --------- | ------ |---------------------------------------------------------|
| LENUM     | Number | The player ID |
| LEPASS    | String | The raw player password |
| IDMP3     | Number | TBD |

No response is required.

### set_server

Create a new server.

This method is used to create a new server, registered under the `DevenirServer`
name.

Payload:
```
LENUM=%i&LEPASS=%s&LESOFT=2&LECOMMENT=%s&MAX_PLAYERS=%i&CFT=%i&CLE_TOURNOIS=%i&ROUND=%i&MD5=%s&DESC=%s&PRIVEE=%s&ARMES=%s
```

Description of the arguments:
| Argument     | Type   | Description                                           |
| ---------    | ------ |-------------------------------------------------------|
| LENUM        | Number | The player ID |
| LEPASS       | String | The raw player password |
| LESOFT       | Number | The software ID |
| LECOMMENT    | String | TBD |
| MAX_PLAYERS  | Number | Number of players |
| CFT          | Number | Gametype, 0 Carnage, 1 CTF, 2 Match, 3 Sniper, 4 Team|
| CLE_TOURNOIS | Number | The internal tournament ID |
| ROUND        | Number | TBD, round number |
| MD5          | String | TBD, may be a MD5 hash of the map |
| DESC         | String | TBD, description of the server |
| PRIVEE       | String | 0 or 1, encoded as a string, is the server private |
| ARMES        | String | Allowed weapons, encoded as a string (see below) |

#### ARMES argument

The `ARMES` argument is encoded as a string. Each character a `*` for a weapon
that's allowed, a `-` for a weapon that's not allowed. The weapons are encoded
into a 29 characters string.

For example, if the string is `*****************************`, allows all of
the weapons in the game. If the string is `-----------------------------`, no
weapons are allowed.

The weapons are encoded in this order from left to right:
| Position | Weapon      |
| -------- | ----------- |
| 1        | USP         |
| 2        | Deagle      |
| 3        | P228        |
| 4        | Colt        |
| 5        | XM1014      |
| 6        | Silver-XM   |
| 7        | Silver-M3   |
| 8        | SPAS        |
| 9        | MAC10       |
| 10       | MP5         |
| 11       | UMP45       |
| 12       | AK-47       |
| 13       | Famas       |
| 14       | M16-A4      |
| 15       | P90         |
| 16       | M249        |
| 17       | G36-552     |
| 18       | SG-552      |
| 19       | SDM-R       |
| 20       | M4          |
| 21       | M-40A4      |
| 22       | AWP         |
| 23       | FR-F2       |
| 24       | Silver-AWP  |
| 25       | fumigene    |
| 26       | grenade     |
| 27       | c4          |
| 28       | plasma      |

#### Response

Response:
```xml
<?xml version="1.0" encoding="ISO-8859-1"?>
<root>
<!-- %i -->
</root>
```

### get_server

Get the list of the current servers.

Payload:
```
LENUM=%i&LEPASS=%s&LESOFT=2&CLE_TOURNOIS=%i&ROUND=%i
```

Description of the arguments:
| Argument     | Type   | Description                                           |
| ---------    | ------ |-------------------------------------------------------|
| LENUM        | Number | The player ID |
| LEPASS       | String | The raw player password |
| LESOFT       | Number | The software ID |
| CLE_TOURNOIS | Number | The internal tournament ID |
| ROUND        | Number | TBD, round number |

Response:
```xml
<?xml version="1.0" encoding="ISO-8859-1"?>
<root>
    <server IP="%s" SERVERID="%i" PLAYERID="%i">
        <NOM>%s</NOM>
        <VERSION>%s</VERSION>
        <COMMENT>%s</COMMENT>
        <MD5>%s</MD5>
        <DESC>%s</DESC>
        <MAP>%s</MAP>
    </server>
    <!-- ... -->
</root>
```

Description of the response:
| Argument  | Type   | Description                                             |
| --------- | ------ |---------------------------------------------------------|
| IP        | String | The server IP address |
| SERVERID  | Number | The server ID |
| PLAYERID  | Number | TBD, the player ID who's hosting the server |
| NOM       | String | The server name |
| VERSION   | String | The server version, setting it to '1' works |
| COMMENT   | String | Server rules |
| MD5       | String | TBD, maybe a MD5 of the map, setting it to "" works |

### get_id

Payload:
```
LELOGIN=%s&LEPASS=%s&LESOFT=2
```

Description of the arguments:
| Argument  | Type   | Description                                             |
| --------- | ------ |---------------------------------------------------------|
| LELOGIN   | String | The player login |
| LEPASS    | String | The raw player password |
| LESOFT    | Number | The software ID |

Response:
```xml
<?xml version="1.0" encoding="ISO-8859-1"?>
<root>
<ID_PLAYER>%i</ID_PLAYER>
</root>
```

Description of the response:
| Argument  | Type   | Description                                             |
| --------- | ------ |---------------------------------------------------------|
| ID_PLAYER | Number | The player ID  returned by the server |

### info_joueur

Get the player informations, such as level and score.

Payload:
```
LENUM=%i&LEPASS=%s&LAMAC=%s
```

Description of the arguments:
| Argument  | Type   | Description                                             |
| --------- | ------ |---------------------------------------------------------|
| LENUM     | Number | The player ID |
| LEPASS    | String | The raw player password |
| LAMAC     | String | The player MAC address |


Response:
```xml
<?xml version="1.0" encoding="ISO-8859-1"?>
<root>
  <NAME>%s</NAME>
  <ERROR>%i</ERROR>
  <MP3 ID="%i">%s</MP3>
  <MODEL>%s</MODEL>
  <IS_OP>%</IS_OP>
  <VALIDE>%i</VALIDE>
  <MSG1>%s</MSG1>
  <MSG2>%s</MSG2>
  <MSG3>%s</MSG3>
  <MSG4>%s</MSG4>
  <SCORE>%i</SCORE>
  <SCROLL>%s</SCROLL>
  <STATS>%s</STATS>
  <PANEL>%s</PANEL>
  <ROMUCHAT>%s</ROMUCHAT>
  <ID_PLAYER>%i</ID_PLAYER>
  <CONTROLE>
    <!-- TBD -->
  </CONTROLE>
</root>
```

Description of the response:
| Argument  | Type   | Description                                             |
| --------- | ------ |---------------------------------------------------------|
| NAME      | String | The player name |
| ERROR     | Number | Internal Error ID |
| MP3       | String | TBD, has a ID |
| MODEL     | String | TBD, model of the player |
| IS_OP     | Number | Is a admin, 0 or 1 |
| VALIDE    | Number | Is the account activated by email, 0 or 1 |
| MSG1      | String | TBD |
| MSG2      | String | TBD |
| MSG3      | String | TBD |
| MSG4      | String | TBD |
| SCORE     | Number | The player score |
| SCROLL    | String | The scrolling text at the bottom of the screen |
| STATS     | String | TBD, the player stats |
| PANEL     | String | The main board in the center of the screen |
| ROMUCHAT  | String | TBD, the chat server |
| ID_PLAYER | Number | The player ID |
| CONTROLE  | String | TBD, the MD5 hash of all of the files for anticheat |

### joinserver

Join a server.

Payload:
```
LENUM=%i&LEPASS=%s&LESOFT=2&SERVERID=%i
```

Description of the arguments:
| Argument  | Type   | Description                                             |
| --------- | ------ |---------------------------------------------------------|
| LENUM     | Number | The player ID |
| LEPASS    | String | The raw player password |
| LESOFT    | Number | The software ID |
| SERVERID  | Number | The server ID |

Response:
```xml
<? xml version="1.0" encoding="ISO-8859-1"?>
<root>%i</root>
```

Description of the response:
| Argument  | Type   | Description                                             |
| --------- | ------ |---------------------------------------------------------|
| %i        | Number | TBD, maybe the game ID that's played |

### get_tournois

Get the list of the current tournaments.

Payload:
```
LENUM=%i&LEPASS=%s&LESOFT=2
```

Description of the arguments:
| Argument  | Type   | Description                                             |
| --------- | ------ |---------------------------------------------------------|
| LENUM     | Number | The player ID |
| LEPASS    | String | The raw player password |
| LESOFT    | Number | The software ID |

Response:
```xml
<?xml version="1.0" encoding="ISO-8859-1"?>
<root>
  <tournois>
    <DESC>%s</DESC>
    <MAP>%s</MAP>
    <ROUND>%i</ROUND>
    <TIMEOUT>%i</TIMEOUT>
  </tournois>
  <!-- ... -->
</root>
```

Description of the response:
| Argument  | Type   | Description                                             |
| --------- | ------ |---------------------------------------------------------|
| DESC      | String | Description of the tournament |
| MAP       | String | The map of the tournament |
| ROUND     | Number | TBD, round number |
| TIMEOUT   | Number | TBD, timeout response or ping |

Please note that the `tournois` element is repeated for each tournament.

### info_tournois

Get a tournament informations.

Payload:
```
LENUM=%i&LEPASS=%s&LESOFT=2&ROUND=%i&CLE_TOURNOIS=%i
```

Description of the arguments:
| Argument     | Type   | Description                                           |
| ---------    | ------ |-------------------------------------------------------|
| LENUM        | Number | The player ID |
| LEPASS       | String | The raw player password |
| LESOFT       | Number | The software ID |
| ROUND        | Number | TBD, round number |
| CLE_TOURNOIS | Number | The internal tournament ID |

Response:
```xml
<?xml version="1.0" encoding="ISO-8859-1"?>
<root>
  <element>
    <DETAIL>%i</DETAIL>
    </element>
  </element>
</root>
```
Note: This may be inaccurate and lack informations.

### set_objet

Set the position a objects in the map.

Payload:
```
LENUM=%i&LEPASS=%s&P[x]=%d&P[y]=%d&P[z]=%d&D[x]=%d&D[y]=%d&D[z]=%d&U[x]=%d&U[y]=%d&U[z]=%d&T=%i&M=%s
```

Description of the arguments:
| Argument  | Type   | Description                                             |
| --------- | ------ |---------------------------------------------------------|
| LENUM     | Number | The player ID |
| LEPASS    | String | The raw player password |
| P[x]      | Number | TBD, x position |
| P[y]      | Number | TBD, y position |
| P[z]      | Number | TBD, z position |
| D[x]      | Number | TBD, x direction |
| D[y]      | Number | TBD, y direction |
| D[z]      | Number | TBD, z direction |
| U[x]      | Number | TBD, x unknown |
| U[y]      | Number | TBD, y unknown |
| U[z]      | Number | TBD, z unknown |
| T         | Number | TBD, object type |
| M         | String | TBD, map name |

No response is required.

### get_objet

Get the position of the objects in the map.

Payload:
```
LENUM=%i&LEPASS=%s&LAMAP=%s
```

Description of the arguments:
| Argument  | Type   | Description                                             |
| --------- | ------ |---------------------------------------------------------|
| LENUM     | Number | The player ID |
| LEPASS    | String | The raw player password |
| LAMAP     | String | The map name |

Response:
```xml
<?xml version="1.0" encoding="ISO-8859-1"?>
<root>
    <OBJ>
        <ID>%i</ID>
        <X>%f</X>
        <Y>%f</Y>
        <Z>%f</Z>
        <DX>%f</DX>
        <DY>%f</DY>
        <DZ>%f</DZ>
        <HX>%f</HX>
        <HY>%f</HY>
        <HZ>%f</HZ>
        <TYPE>%i</TYPE>
    </OBJ>
    <!-- ... -->
</root>
```

Description of the response:
| Argument  | Type   | Description                                             |
| --------- | ------ |---------------------------------------------------------|
| ID        | Number | TBD, object ID |
| X         | Float  | X position |
| Y         | Float  | Y position |
| Z         | Float  | Z position |
| DX        | Float  | X direction |
| DY        | Float  | Y direction |
| DZ        | Float  | Z direction |
| HX        | Float  | TBD, X unknown |
| HY        | Float  | TBD, Y unknown |
| HZ        | Float  | TBD, Z unknown |
| TYPE      | Number | TBD, object type. The game check if TYPE > 0 |

This one looks complicated, and may take a while to document. It returns a
lot of informations about the map probably.

