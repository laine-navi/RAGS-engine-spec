# RAGS Engine Specification

Reverse-engineered specification of the RAGS game engine, derived from decompiled
source (ilspycmd 11.x). Documents the SDF database schema, text substitution system,
command/condition enums, and AES encryption used in the file format.

Based on RAGS Suite 3.0.60 (WPF). The command and condition system is identical
across all versions from 2.5.2 through 3.0.60 (verified by enum diff between 2.6.1
and 3.0.60). The 3.0 release was a UI framework migration (WinForms → WPF) and
file format change (NRBF .NET BinaryFormatter → SQL Server Compact 3.5 SDF database),
not an engine feature addition. All 103 command types and 37 condition types were
present in 2.6.1.

**Preservation archive:** [rags game on archive.org](https://archive.org/details/rags-suite-preservation-archive)

**Decompiled source:** [RAGS-decompiled](https://github.com/laine-navi/RAGS-decompiled)

---

## 1. File Format

### 1.1 Container

The `.rags` file is a **SQL Server Compact 3.5** database (`.sdf` binary format),
encrypted at the file level.

- **SQL CE encryption password:** `Ç°¥àòÅÅÇÉàññø`
- **Max database size:** 4024 KB
- **Connection string:** `DataSource=<path>;Max Database Size=4024;Password=Ç°¥àòÅÅÇÉàññø`

### 1.2 Application-Level Encryption (AES-256-CBC)

The `Data ntext` columns in `*Actions` tables contain AES-encrypted
SharpSerializer binary blobs (serialized `List<Action>`).

**Key derivation:**

```
password = "F1$asDDFHappy"

Step 1: RagsZone XOR transform (each char XOR 129)
  transformed = ''.join(chr(ord(c) ^ 129) for c in password)

Step 2: Key derivation (PBKDF2-SHA1, 2000 iterations)
  salt = b"Ivan Medvedev"  # 13 bytes: [73,118,97,110,32,77,101,100,118,101,100,101,118]
  key_material = PBKDF2(SHA1, transformed.encode('utf-16-le'), salt, 2000, 48)
  aes_key = key_material[:32]   # 256-bit key
  aes_iv  = key_material[32:48] # 16-byte IV

Step 3: AES-256-CBC decrypt
  plaintext = AES_CBC_decrypt(aes_key, aes_iv, ciphertext)
```

**Python equivalent:**

```python
import hashlib
from Crypto.Cipher import AES

def rags_zone(data: str) -> str:
    return ''.join(chr(ord(c) ^ 129) for c in data)

def derive_keys():
    pw = rags_zone("F1$asDDFHappy").encode('utf-16-le')
    km = hashlib.pbkdf2_hmac('sha1', pw, b"Ivan Medvedev", 2000, 48)
    return km[:32], km[32:48]

def decrypt_actions(ciphertext: bytes) -> bytes:
    key, iv = derive_keys()
    return AES.new(key, AES.MODE_CBC, iv).decrypt(ciphertext)
```

Note: .NET's `Rijndael.Create()` with a 32-byte key = AES-256, CBC mode (IV provided).

Note: `EncDec.RagsZone("F1$asDDFHappy")` is called in `GetDBConnection()` with
the return value discarded. It is a pure function with no side effects — this call
is a no-op (likely dead code or a decompiler artifact).

### 1.3 Embedded Media

The `Media` table stores raw image/video binary data in a `Data image` column.
The `.rags` file is fully self-contained — no external asset files needed.

### 1.4 Save Games

Game state is serialized via **SharpSerializer** (binary mode), then AES-encrypted
with the same key derivation as §1.2.

### 1.5 What 3.0 Changed (vs 2.x)

The engine logic (PerformTextReplacements, command dispatch, condition evaluation)
is the same codebase. The 3.0 release changed:

1. **UI framework:** WinForms → WPF (new XAML-based windows, WebBrowser control
   for content rendering instead of WinForms RichTextBox)
2. **File format:** NRBF (.NET BinaryFormatter) → SQL Server Compact 3.5 SDF
3. **File-level encryption:** SQL CE database encryption (password in connection string)
4. **Media storage:** Inline `image` BLOB column in the Media table

---

## 2. Database Schema

### 2.1 GameData (single row)

```sql
CREATE TABLE GameData(
    Title nvarchar(250),
    OpeningMessage ntext,
    HideMainPicDisplay bit,
    UseInlineImages bit,
    HidePortrait bit,
    AuthorName nvarchar(250),
    GameVersion nvarchar(50),
    GameInformation ntext,
    bgMusic nvarchar(250),
    RepeatbgMusic bit,
    PasswordProtected bit,
    GamePassword nvarchar(250),
    ObjectVersionNumber nvarchar(250),
    GameFont nvarchar(250),
    -- added by 3.0 upgrade:
    RoomGroups ntext,
    ClothingZoneLevels ntext,
    NotificationsOff bit NOT NULL DEFAULT(0)
);
```

### 2.2 Rooms

```sql
CREATE TABLE Rooms(
    UniqueID nvarchar(250),
    Description ntext,
    SDesc nvarchar(250),
    Name nvarchar(250),
    RoomPic nvarchar(250),
    EnterFirstTime bit,
    LeaveFirstTime bit,
    [Group] nvarchar(250),
    LayeredRoomPic nvarchar(250)
);
```

### 2.3 RoomExits

```sql
CREATE TABLE RoomExits (
    ID int IDENTITY(1,1) NOT NULL,
    RoomID nvarchar(250),
    Direction int,
    Active bit,
    DestinationRoom nvarchar(250),
    PortalObjectName nvarchar(250)
);
```

### 2.4 Items

```sql
CREATE TABLE Items(
    UniqueID nvarchar(250),
    Name nvarchar(250),
    Description ntext,
    SDesc nvarchar(250),
    Preposition nvarchar(250),
    LocationName nvarchar(250),
    LocationType int,
    Carryable bit,
    Wearable bit,
    Openable bit,
    Lockable bit,
    Enterable bit,
    Readable bit,
    Container bit,
    Weight float,
    Worn bit,
    [Read] bit,
    Locked bit,
    [Open] bit,
    Entered bit,
    Visible bit,
    EnterFirstTime bit,
    LeaveFirstTime bit
    -- added by 3.0 upgrade:
    -- GroupName nvarchar(255)
    -- Important bit NOT NULL DEFAULT(0)
);
```

### 2.5 Characters

```sql
CREATE TABLE Characters(
    Charname nvarchar(250),
    CharnameOverride nvarchar(250),
    CharGender int,
    CurrentRoom nvarchar(250),
    Description ntext,
    AllowInventoryInteraction bit,
    EnterFirstTime bit,
    LeaveFirstTime bit
);
```

### 2.6 Variables

```sql
CREATE TABLE Variables(
    VarName nvarchar(250),
    String ntext,
    NumType float,
    Min nvarchar(50),
    Max nvarchar(50),
    EnforceRestrictions bit,
    VarComment ntext,
    dtDateTime datetime,
    VarType int,
    VarArray ntext
);
```

### 2.7 Timer

```sql
CREATE TABLE Timer(
    Name nvarchar(250),
    TType int,
    Active bit,
    Restart bit,
    TurnNumber int,
    Length int,
    LiveTimer bit,
    TimerSeconds int
);
```

### 2.8 Player (single row)

```sql
CREATE TABLE Player(
    Name nvarchar(250),
    Description ntext,
    StartingRoom nvarchar(250),
    CurrentRoom nvarchar(250),
    PlayerLayeredImage nvarchar(250),
    PlayerGender int,
    PromptForName bit,
    PromptForGender bit,
    PlayerPortrait nvarchar(250),
    EnforceWeight bit,
    WeightLimit float
);
```

### 2.9 Media

```sql
CREATE TABLE Media(
    Name nvarchar(250),
    BackgroundColor nvarchar(250),
    TextColor nvarchar(250),
    TextFont nvarchar(250),
    ImageName nvarchar(250),
    UseEnhancedGraphics bit,
    NewImage nvarchar(250),
    Data image
);
```

### 2.10 MediaGroups

```sql
CREATE TABLE MediaGroups (
    ID int IDENTITY(1,1) NOT NULL,
    Name nvarchar(255),
    Parent nvarchar(255)
);
```

### 2.11 Child Tables (Actions & Properties)

Each entity type has an `*Actions` table (AES-encrypted SharpSerializer blob
containing `List<Action>`) and an `*Properties` table (key-value custom properties).

```sql
CREATE TABLE RoomActions (
    ID int IDENTITY(1,1) NOT NULL,
    RoomID nvarchar(250),
    Data ntext
);

CREATE TABLE RoomProperties (
    ID int IDENTITY(1,1) NOT NULL,
    RoomID nvarchar(250),
    Name nvarchar(250),
    Value ntext
);

CREATE TABLE ItemActions (
    ID int IDENTITY(1,1) NOT NULL,
    ItemID nvarchar(250),
    Data ntext
);

CREATE TABLE ItemProperties (
    ID int IDENTITY(1,1) NOT NULL,
    ItemID nvarchar(250),
    Name nvarchar(250),
    Value ntext
);

CREATE TABLE ItemLayeredZoneLevels (
    ID int IDENTITY(1,1) NOT NULL,
    ItemID nvarchar(250),
    Data nvarchar(250)
);

CREATE TABLE CharacterActions (
    ID int IDENTITY(1,1) NOT NULL,
    Charname nvarchar(250),
    Data ntext
);

CREATE TABLE CharacterProperties (
    ID int IDENTITY(1,1) NOT NULL,
    Charname nvarchar(250),
    Name nvarchar(250),
    Value ntext
);

CREATE TABLE VariableProperties (
    ID int IDENTITY(1,1) NOT NULL,
    VarName nvarchar(250),
    Name nvarchar(250),
    Value ntext
);

CREATE TABLE TimerActions (
    ID int IDENTITY(1,1) NOT NULL,
    Name nvarchar(250),
    Data ntext
);

CREATE TABLE TimerProperties (
    ID int IDENTITY(1,1) NOT NULL,
    TimerName nvarchar(250),
    Name nvarchar(250),
    Value ntext
);

CREATE TABLE PlayerActions (
    ID int IDENTITY(1,1) NOT NULL,
    Data ntext
);

CREATE TABLE PlayerProperties (
    ID int IDENTITY(1,1) NOT NULL,
    Name nvarchar(250),
    Value ntext
);

CREATE TABLE MediaLayeredImages (
    ID int IDENTITY(1,1) NOT NULL,
    MediaName nvarchar(250),
    Data nvarchar(250)
);
```

---

## 3. Object Model

```
Game
├── ThePlayer: Player
│   ├── Name, Description, CurrentRoom, StartingRoom
│   ├── PlayerGender, PlayerPortrait, PlayerLayeredImage
│   ├── EnforceWeight, WeightLimit
│   ├── Actions: List<Action>
│   └── CustomProperties: List<CustomProperty>
├── TheRooms: List<Room>
│   ├── UniqueID (GUID), Name, SDesc, Description
│   ├── RoomPic, LayeredRoomPic, [Group]
│   ├── Exits: List<Exit>
│   ├── Actions: List<Action>
│   └── CustomProperties: List<CustomProperty>
├── TheObjects: List<GameObject>  (from Items table)
│   ├── UniqueID (GUID), Name, SDesc, Description
│   ├── Preposition, LocationName, LocationType
│   ├── Carryable, Wearable, Openable, Lockable, Enterable, Readable
│   ├── Container, Weight, Worn, [Read], Locked, [Open], Entered, Visible
│   ├── EnterFirstTime, LeaveFirstTime
│   ├── Actions: List<Action>
│   └── CustomProperties: List<CustomProperty>
├── TheCharacters: List<Character>
│   ├── Charname, CharnameOverride, CharGender
│   ├── CurrentRoom, Description
│   ├── AllowInventoryInteraction, EnterFirstTime, LeaveFirstTime
│   ├── Actions: List<Action>
│   └── CustomProperties: List<CustomProperty>
├── TheVariables: List<GameVariable>
│   ├── VarName, VarType, NumType, String, dtDateTime
│   ├── Min, Max, EnforceRestrictions, VarComment
│   ├── VarArray (deserialized from ntext)
│   └── CustomProperties: List<CustomProperty>
├── TheTimers: List<Timer>
│   ├── Name, TType, Active, Restart, TurnNumber, Length, LiveTimer, TimerSeconds
│   ├── Actions: List<Action>
│   └── CustomProperties: List<CustomProperty>
├── ThePictures: List<GamePicture>  (from Media table)
│   ├── Name, BackgroundColor, TextColor, TextFont
│   ├── ImageName, UseEnhancedGraphics, NewImage
│   ├── Data (raw binary)
│   └── LayeredImages: List<string>
├── TheStatusBarItems: List<StatusBarItem>
├── QuickCommands: List<QuickMenuItem>
├── TheContent: StringBuilder (the HTML output buffer)
└── SortOrder: SO_NATURAL | SO_ALPHA | SO_LILO
```

### 3.1 LocationType

```
LT_NULL
LT_IN_OBJECT       // inside a container
LT_ON_OBJECT       // on top of an object
LT_ROOM
LT_PLAYER
LT_CHARACTER
LT_PORTAL          // referenced by RoomExits.PortalObjectName
```

### 3.2 Action

```
Action
├── actdata: GameObject
├── name: string
├── bActive: bool
├── overridename: string
├── actionparent: string
├── bConditionFailOnFirst: bool
├── InputType: ActionInputType
│   None | Object | Character | ObjectOrCharacter | Text | Custom | Inventory
├── PassCommands: List<object>
├── FailCommands: List<object>
├── Conditions: List<Condition>
├── CustomChoices: List<string>
├── EnhInputData: EnhancedInputData
└── CustomChoiceTitle: string
```

### 3.3 CustomProperty

```
CustomProperty
├── Name: string
└── Value: string
```

---

## 4. Text Substitution Engine

### 4.1 Algorithm

```python
def perform_text_replacements(text: str, loopobject=None, newvalue: str = "") -> str:
    """
    Loop until no more substitutions are found.
    Scan RIGHT-TO-LEFT for '[' tokens.
    """
    changed = True
    while changed:
        changed = False
        idx = text.rfind('[')
        while idx >= 0:
            end = text.find(']', idx)
            if end == -1:
                idx = text.rfind('[', 0, idx)
                continue
            tag = text[idx:end+1].upper()
            colon = tag.find(':')
            if colon != -1:
                prefix = tag[:colon+1]  # e.g. "[V:", "[RP:", etc.
                if prefix in KNOWN_TAGS:
                    substitute(text, idx, end, prefix, loopobject, newvalue)
                    changed = True
                else:
                    changed = False
            elif is_html_format_tag(tag):
                # [B], [/B], [I], [/I], [U], [/U], [F color], [C color], [MIDDLE]
                pass  # skip formatting tags
            else:
                # Generic [TAG] resolved against loopobject
                substitute_generic(text, idx, end, tag, loopobject)
            idx = text.rfind('[', 0, idx)
    return text
```

**Critical:** Loops until a full pass produces no changes (handles nested tokens).
Scan is right-to-left (`LastIndexOf`).

### 4.2 Tag Reference

| Tag               | Resolves to                             | Notes                                                  |
|-------------------|-----------------------------------------|--------------------------------------------------------|
| `[V:name]`        | Variable value by name                  | Replaced with `newvalue` param if provided             |
| `[RP:room:prop]`  | Room custom property                    | `[RP:name:prop]` or `[RP:(GUID):prop]`                 |
| `[IP:item:prop]`  | Item custom property                    | Same format as RP                                      |
| `[PP:prop]`       | Player custom property                  | `[PP:propname]`                                        |
| `[CP:char:prop]`  | Character custom property               |                                                        |
| `[VP:var:prop]`   | Variable custom property                |                                                        |
| `[TP:timer:prop]` | Timer custom property                   |                                                        |
| `[IA:item]`       | Item array property (loop context)      | Resolved against `loopobject`                          |
| `[TA:timer]`      | Timer array property (loop context)     |                                                        |
| `[CA:char]`       | Character array property (loop context) |                                                        |
| `[RA:room]`       | Room array property (loop context)      |                                                        |
| `[PA:player]`     | Player array property (loop context)    |                                                        |
| `[JSA:varname]`   | Variable value, `\r`→`\\r`, `\n`→`\\n`  | For embedding in JS string literals. NOT JS execution. |
| `[B]` / `[/B]`    | `<b>` / `</b>`                          | Formatting, not substitution                           |
| `[I]` / `[/I]`    | `<i>` / `</i>`                          |                                                        |
| `[U]` / `[/U]`    | `<u>` / `</u>`                          |                                                        |
| `[F color]`       | Font color                              | `[F #FF0000]`                                          |
| `[C color]`       | Color (alt)                             |                                                        |
| `[MIDDLE]`        | Vertical center alignment               |                                                        |

### 4.3 Loop Context

When called from a `CT_Loop_*` command handler, `loopobject` is the current entity
being iterated (a `GameObject`, `Character`, `Room`, `Exit`, or `Timer`). The
generic `[TAG]` handler (no colon) resolves property names against this object.
The `[IA:]`, `[CA:]`, etc. tags also use this context.

When `loopobject` is `null` (the common case), only the colon-prefixed tags resolve.

### 4.4 Built-in JavaScript (InitialJavaScript)

Injected into the HTML content area on game load. Available to game scripts that
use `CT_JAVA_SET_RAGS` or `CT_VARIABLE_SET_JAVASCRIPT`:

```javascript
function getVerticalScrollPosition() { return document.body.scrollTop.toString(); }
function setVerticalScrollPosition(position) { document.body.scrollTop = position; }
function pagedown() { document.body.scrollTop += document.body.clientHeight; }
function atbottom() {
    return (document.body.scrollTop + document.body.clientHeight) >= document.body.scrollHeight;
}
```

---

## 5. Command System

### 5.1 Structure

Each command row (stored in the encrypted `Data` blob) has:

| Field           | Type              | Purpose                                                |
|-----------------|-------------------|--------------------------------------------------------|
| `cmdtype`       | CommandType enum  | What the command does                                  |
| `CommandText`   | string            | Primary parameter (often a SQL query or variable name) |
| `CommandPart2`  | string            | Secondary parameter                                    |
| `CommandPart3`  | string            | Tertiary parameter                                     |
| `CommandPart4`  | string            | Quaternary parameter                                   |
| `CustomChoices` | List\<string\>    | For input commands                                     |
| `EnhInputData`  | EnhancedInputData | Enhanced input dialog config                           |

All text fields go through `PerformTextReplacements` before use.

### 5.2 CT_Execute_SQL

The generic SQL executor:

```
text  = PerformTextReplacements(CommandPart2)  → target variable name
text4 = PerformTextReplacements(CommandText)   → SQL query

1. Look up variable by name
2. Open SQL CE connection
3. Execute query
4. Store result set into variable.VarArray as List<List<string>>
```

### 5.3 Full CommandType Enum (103 entries)

```
CT_UNINITIALIZED
CT_ACTION_ADD_CUSTOMCHOICE
CT_ACTION_CLEAR_CUSTOMCHOICE
CT_ACTION_REMOVE_CUSTOMCHOICE
CT_DISPLAYCHARDESC
CT_CHAR_DISPLAYPORT
CT_MOVECHAR
CT_MOVECHARINVTOPLAYER
CT_CHAR_MOVETOOBJ
CT_CHAR_SETPORT
CT_SETCHARACTION
CT_CHAR_SET_CUSTOM_PROPERTY
CT_CHAR_SET_CUSTOM_PROPERTY_JS
CT_SETCHARDESC
CT_CHAR_SET_GENDER
CT_CHAR_SET_NAME
CT_COMMENT
CT_DEBUGTEXT
CT_JAVA_SET_RAGS
CT_DISPLAYTEXT
CT_EXPORTVARIABLE
CT_IMPORTVARIABLE
CT_PAUSEGAME
CT_DISPLAYLAYEREDPICTURE
CT_DISPLAYPICTURE
CT_MM_SET_BACKGROUND_MUSIC
CT_MM_STOP_BACKGROUND_MUSIC
CT_MM_PLAY_SOUNDEFFECT
CT_MM_SET_MAIN_COMPASS
CT_MM_SET_UD_COMPASS
CT_LAYEREDIMAGE_ADD
CT_LAYEREDIMAGE_CLEAR
CT_LAYEREDIMAGE_REMOVE
CT_LAYEREDIMAGE_REPLACE
CT_DISPLAYITEMDESC
CT_ITEM_LAYERED_REMOVE
CT_ITEM_LAYERED_WEAR
CT_MOVEITEMTOCHAR
CT_MOVEITEMTOINV
CT_MOVEITEMTOOBJ
CT_MOVEITEMTOROOM
CT_SETOBJECTACTION
CT_ITEM_SET_CUSTOM_PROPERTY
CT_ITEM_SET_CUSTOM_PROPERTY_JS
CT_SETITEMDESC
CT_ITEM_SET_NAME_OVERRIDE
CT_SETLOCKEDUNLOCKED
CT_SETOPENCLOSED
CT_SETITEMTOWORN
CT_ITEM_SET_VISIBILITY
CT_ITEMS_MOVE_CONTAINER_TO_CONTAINER
CT_LOOP_BREAK
CT_CANCELMOVE
CT_DISPLAYPLAYERDESC
CT_SETLAYEREDPLAYERPORTRAIT
CT_MOVEINVENTORYTOCHAR
CT_MOVEINVENTORYTOROOM
CT_MOVEPLAYER
CT_MOVETOCHAR
CT_MOVETOOBJ
CT_PLAYER_SET_CUSTOM_PROPERTY
CT_PLAYER_SET_CUSTOM_PROPERTY_JS
CT_SETPLAYERACTION
CT_SETPLAYERDESC
CT_SETPLAYERGENDER
CT_SETPLAYERNAME
CT_SETPLAYERPORTRAIT
CT_DISPLAYROOMDESCRIPTION
CT_DISPLAYROOMPICTURE
CT_ROOM_MOVE_ITEMS_TO_PLAYER
CT_SETROOMACTION
CT_ROOM_SET_CUSTOM_PROPERTY
CT_ROOM_SET_CUSTOM_PROPERTY_JS
CT_SETROOMDESCRIPTION
CT_SETEXIT
CT_SETEXITDESTINATION
CT_ROOM_SET_NAME_OVERRIDE
CT_SETROOMLAYEREDPIC
CT_SETROOMPIC
CT_Execute_SQL
CT_Status_ItemVisibleInvisible
CT_EXECUTETIMER
CT_RESETTIMER
CT_TIMER_SET_CUSTOM_PROPERTY
CT_TIMER_SET_CUSTOM_PROPERTY_JS
CT_SETTIMER
CT_DISPLAYVARIABLE
CT_SETVARIABLE
CT_ITEM_GETRANDOMGROUP
CT_MM_GETRANDOMGROUP
CT_VARIABLE_SET_JAVASCRIPT
CT_SETVARIABLEBYINPUT
CT_SETVARIABLE_NUMERIC_BYINPUT
CT_VARIABLE_SET_CUSTOM_PROPERTY
CT_VARIABLE_SET_CUSTOM_PROPERTY_JS
CT_VARIABLE_SET_RANDOMLY
CT_VARIABLE_SET_WITH_CHARPROPERTYVALUE
CT_VARIABLE_SET_WITH_ITEMPROPERTYVALUE
CT_VARIABLE_SET_WITH_PLAYERPROPERTYVALUE
CT_VARIABLE_SET_WITH_ROOMPROPERTYVALUE
CT_VARIABLE_SET_WITH_TIMERPROPERTYVALUE
CT_VARIABLE_SET_WITH_VARIABLE
CT_VARIABLE_SET_WITH_VARIABLEPROPERTYVALUE
CT_ENDGAME
```

### 5.4 JavaScript Command Types

`CT_VARIABLE_SET_JAVASCRIPT`, `CT_JAVA_SET_RAGS`, and the six
`CT_*_SET_CUSTOM_PROPERTY_JS` variants execute JavaScript code against the WPF
`WebBrowser` control (MSHTML/IE engine). The JS can:

- Call the built-in functions (`getVerticalScrollPosition`, etc.)
- Manipulate the DOM of the game's content area
- Set RAGS variables (via a bridge function)

For rags2html: these need either a JS sandbox (e.g., QuickJS compiled to WASM)
or at minimum a no-op with a console warning.

---

## 6. Condition System

### 6.1 Structure

```
Condition
├── conditionname: string
├── Checks: List<Check>
├── PassCommands: List<Command>
└── FailCommands: List<Command>

Check
├── CondType: ConditionType (37 values)
├── CkType: CheckType (And | Or)  ← how this check combines with others in the Condition
├── ConditionStep2: string  (resolved via PerformTextReplacements)
├── ConditionStep3: string  (resolved via PerformTextReplacements)
└── ConditionStep4: string  (resolved via PerformTextReplacements; often the comparison operator)
```

Note: `bConditionFailOnFirst` lives on the parent `Action`, not on `Condition`
or `Check`. It controls whether the action stops evaluating after the first
failed condition.

### 6.2 CheckType (logical combinator)

```
CT_Uninitialized
And   → all Checks in the Condition must pass
Or    → any Check in the Condition may pass
```

Comparison operators (equals, greater than, contains, etc.) are string values
in `ConditionStep4`, not an enum.

### 6.3 Full ConditionType Enum (37 entries)

```
CT_Uninitialized
CT_AdditionalDataCheck
CT_Character_CustomPropertyCheck
CT_Character_Gender
CT_Character_In_Room
CT_Character_In_RoomGroup
CT_Item_CustomPropertyCheck
CT_Item_InGroup
CT_Item_Held_By_Character
CT_Item_Held_By_Player
CT_Item_In_Object
CT_Item_In_Room
CT_Item_In_RoomGroup
CT_Item_Not_Held_By_Player
CT_Item_Not_In_Object
CT_Item_State_Check
CT_Loop_While
CT_Loop_Characters
CT_Loop_Items
CT_Loop_Item_Group
CT_Loop_Item_Container
CT_Loop_Item_Room
CT_Loop_Item_Inventory
CT_Loop_Item_Char_Inventory
CT_Loop_Rooms
CT_Loop_Exits
CT_MultiMedia_InGroup
CT_Player_CustomPropertyCheck
CT_Player_Gender
CT_Player_In_Room
CT_Player_In_RoomGroup
CT_Player_In_Same_Room_As
CT_Player_Moving
CT_Room_CustomPropertyCheck
CT_Timer_CustomPropertyCheck
CT_Variable_Comparison
CT_Variable_To_Variable_Comparison
CT_Variable_CustomPropertyCheck
```

### 6.4 ConditionType → Step Interpretation

| CondType                           | Step2           | Step3            | Step4                              |
|------------------------------------|-----------------|------------------|------------------------------------|
| CT_Item_Held_By_Character          | Character name  | Item name        | —                                  |
| CT_Item_Held_By_Player             | —               | Item name        | —                                  |
| CT_Item_In_Object                  | Object name     | Item name        | —                                  |
| CT_Item_In_Room                    | Room name       | Item name        | —                                  |
| CT_Player_In_Room                  | Room name       | —                | —                                  |
| CT_Player_In_Same_Room_As          | Character name  | —                | —                                  |
| CT_Variable_Comparison             | Variable name   | Comparison value | Operator (=, >, <, Contains, etc.) |
| CT_Variable_To_Variable_Comparison | Variable 1 name | Variable 2 name  | Operator                           |
| CT_*_CustomPropertyCheck           | Entity name     | Property name    | Expected value / Operator          |

All step values go through `PerformTextReplacements` before comparison.

### 6.5 Loop Conditions

`CT_Loop_*` types iterate over a collection and re-evaluate the condition for
each element. The current element is passed as `loopobject` to
`PerformTextReplacements` for all nested commands/conditions.

---

## 7. GameVariable

```
GameVariable
├── VarName: string
├── VarType: enum
│   VT_UNINITIALIZED
│   VT_NUMBER
│   VT_STRING
│   VT_DATETIME
│   VT_NUMBERARRAY
│   VT_STRINGARRAY
│   VT_DATETIMEARRAY
├── NumType: double           (for VT_NUMBER)
├── String: string            (for VT_STRING)
├── dtDateTime: DateTime      (for VT_DATETIME)
├── Min: string (nvarchar(50), may contain [V:] tags)
├── Max: string (nvarchar(50))
├── EnforceRestrictions: bool
├── VarComment: string
├── VarArray: List<object>    (for array types: List<double>, List<string>, List<DateTime>)
└── CustomProperties: List<CustomProperty>
```

---

## 8. Assembly Layout

| Assembly              | Namespace        | Role                                                                                         |
|-----------------------|------------------|----------------------------------------------------------------------------------------------|
| `Rags.exe`            | RAGSWPFTEST1     | WPF shell, command dispatch switch                                                           |
| `RagsSvPlayerLib.dll` | Rags             | Core engine: Game, Command, Condition, Check, GameObject, Character, PerformTextReplacements |
| `WPFRagsLib.dll`      | WPFRagsLib, Rags | SaveLoad (SDF I/O), EncDec (AES), WPF controls                                               |
| `RagsDesigner.exe`    | WPFRagsDesigner  | Designer app (not needed for playback)                                                       |

---

## 9. Known Gaps in rags2html

These command types exist in all versions from 2.5.2 onward (confirmed identical
between 2.6.1 and 3.0.60). Regalia was built around the 2.x "Save To Web" export
which may have excluded or stubbed some of these. Games that break in rags2html
are likely failing on the SDF binary format differences (B-TREE/index handling)
or the SharpSerializer blob format, not on unsupported command types per se.

| Feature                                                    | Severity | Notes                                        |
|------------------------------------------------------------|----------|----------------------------------------------|
| `CT_VARIABLE_SET_JAVASCRIPT`                               | **High** | Needs JS sandbox or explicit no-op + warning |
| `CT_JAVA_SET_RAGS`                                         | **High** | Same                                         |
| `CT_*_SET_CUSTOM_PROPERTY_JS` (×6)                         | **High** | Same                                         |
| `CT_LAYEREDIMAGE_*` (×4)                                   | Medium   | Layered image compositing                    |
| `CT_MM_*` (×5)                                             | Medium   | Music/sound management                       |
| `CT_SETROOMLAYEREDPIC`                                     | Medium   |                                              |
| `CT_SETLAYEREDPLAYERPORTRAIT`                              | Medium   |                                              |
| `CT_ITEM_LAYERED_WEAR` / `CT_ITEM_LAYERED_REMOVE`          | Medium   | Layered clothing                             |
| `CT_VARIABLE_SET_WITH_*PROPERTYVALUE` (×6)                 | Medium   | Read entity property into variable           |
| `CT_ITEM_GETRANDOMGROUP` / `CT_MM_GETRANDOMGROUP`          | Low      | Random selection from groups                 |
| `CT_EXPORTVARIABLE` / `CT_IMPORTVARIABLE`                  | Low      | Variable import/export                       |
| `CT_SETVARIABLEBYINPUT` / `CT_SETVARIABLE_NUMERIC_BYINPUT` | Low      | User input dialogs                           |
| `CT_VARIABLE_SET_RANDOMLY`                                 | Low      | Random variable assignment                   |
| `CT_PAUSEGAME` / `CT_CANCELMOVE` / `CT_LOOP_BREAK`         | Low      |                                              |
| `CT_ENDGAME`                                               | Low      | End game screen                              |
| `CT_COMMENT` / `CT_DEBUGTEXT`                              | None     | No-op / debug                                |

---

## 10. Quick Reference: Opening a .rags File in Python

```python
import hashlib
from Crypto.Cipher import AES

SQLCE_PASSWORD = "Ç°¥àòÅÅÇÉàññø"
AES_PASSWORD = "F1$asDDFHappy"
SALT = b"Ivan Medvedev"
XOR_KEY = 129

def rags_zone(data: str) -> str:
    return ''.join(chr(ord(c) ^ XOR_KEY) for c in data)

def derive_keys():
    pw = rags_zone(AES_PASSWORD).encode('utf-16-le')
    km = hashlib.pbkdf2_hmac('sha1', pw, SALT, 2000, 48)
    return km[:32], km[32:48]

def decrypt_actions(ciphertext: bytes) -> bytes:
    key, iv = derive_keys()
    return AES.new(key, AES.MODE_CBC, iv).decrypt(ciphertext)

# To open the SDF database:
#   - Use a Python SQL CE driver (pysqlce, if available)
#   - Or parse the SDF binary format directly (what rags2html does)
#
# The file is encrypted with SQLCE_PASSWORD at the SQL CE level.
# The application-level AES (above) is for the *Actions Data ntext columns only.
```

---

## 11. Sources

All information derived from decompilation of RAGS Suite 3.0.60:

| File                                             | Key content                                             |
|--------------------------------------------------|---------------------------------------------------------|
| `Rags.exe` → `RAGSWPFTEST1/MainWindow.xaml.cs`   | Command dispatch switch, 103 case handlers              |
| `RagsSvPlayerLib.dll` → `Rags/Game.cs`           | PerformTextReplacements (line 2898), Game class fields  |
| `RagsSvPlayerLib.dll` → `Rags/Command.cs`        | CommandType enum, Command class fields                  |
| `RagsSvPlayerLib.dll` → `Rags/Condition.cs`      | ConditionType enum, Condition class                     |
| `RagsSvPlayerLib.dll` → `Rags/Check.cs`          | Check class, ValidateCheck                              |
| `RagsSvPlayerLib.dll` → `Rags/CheckType.cs`      | CheckType enum (And/Or)                                 |
| `RagsSvPlayerLib.dll` → `Rags/GameVariable.cs`   | VarType enum, GameVariable fields                       |
| `RagsSvPlayerLib.dll` → `Rags/GameObject.cs`     | LocationType enum                                       |
| `RagsSvPlayerLib.dll` → `Rags/Action.cs`         | ActionInputType enum, Action class                      |
| `RagsSvPlayerLib.dll` → `Rags/CustomProperty.cs` | CustomProperty class                                    |
| `WPFRagsLib.dll` → `WPFRagsLib/SaveLoad.cs`      | GetDBConnection, CREATE TABLE statements, Game_LoadGame |
| `WPFRagsLib.dll` → `Rags/EncDec.cs`              | RagsZone (XOR 129), AES key derivation, salt            |
