# ULog 文件格式

ULog 是用于记录系统数据的文件格式。 格式是自描述的，即它包含记录的格式和消息类型。

它可用于记录设备输入（传感器等）、内部状态（cpu 负载、姿态等）以及打印的日志消息。

这种格式对所有的二进制类型采用小端模式。

## 数据类型

使用以下二进制类型。 它们都对应 C 语言中的类型：

| 类型                  | 大小（以字节为单位） |
| ------------------- | ---------- |
| int8_t, uint8_t   | 1          |
| int16_t, uint16_t | 2          |
| int32_t, uint32_t | 4          |
| int64_t, uint64_t | 8          |
| float               | 4          |
| double              | 8          |
| bool, char          | 1          |

此外，所有的类型还可以作为数组使用，比如 `float[5]`。 通常，所有的字符串（`char[length]`）在末尾不包含 `'\0'`。 字符串比较区分大小写。

## 文件结构

该文件由三个部分组成：

    ----------------------
    |         头         |
    ----------------------
    |        定义        |
    ----------------------
    |        数据        |
    ----------------------
    

### 头部分

头是一个固定大小的部分，具有以下格式（16个字节）：

    ----------------------------------------------------------------------
    | 0x55 0x4c 0x6f 0x67 0x01 0x12 0x35 | 0x01         | uint64_t       |
    | File magic(7B)                     | Version (1B) |  Timestamp (8B) |
    ----------------------------------------------------------------------
    

Version 是文件的格式的版本，目前是 1。 Timestamp 是一个 `uint64_t` 的整数，表示从日志开始记录的微秒数。

### 定义部分

可变长度部分，包含版本信息、格式定义和 (初始) 参数值。

定义和数据部分由消息流组成。 每个数据流包含此标头：

    struct message_header_s {
        uint16_t msg_size;
        uint8_t msg_type
    };
    

`msg_size` 是除头 (`hdr_size`= 3 bytes) 外消息的字节大小。 `msg_type` 定义内容类型，是以下的一种：

- 'B' ：比特消息标志

    struct ulog_message_flag_bits_s {
        uint8_t compat_flags[8];
        uint8_t incompat_flags[8];
        uint64_t appended_offsets[3]; ///< file offset(s) for appended data if appending bit is set
    };
    

这条消息**必须**是头后面的第一条消息，这样才有固定的常数偏移量。

- `compat_flags`: 兼容的标志位。 它们目前都没有定义，都必须设置为 0。 这些位可用于将来的 Ulog 更改，即与现有解析器兼容。 这意味着, 如果设置了一个未知位，解析器就可以忽略。
- `incompat_flags`: 不兼容的标志位。 如果日志包含附加数据，并且至少有一个 `appended_offset` 是非零的，那么索引 0 的 LSB 位被设置为 1。 其他位都是未定义的，必须将设置为 0。 如果解析器发现这些位置 1，它必须拒绝解析日志。 这可用于引入现有解析器无法处理的重大更改。
- `appended_offsets`: 附加数据的文件偏移量 (基于 0)。 如果没有附加数据，则所有偏移量必须为零。 这可以用于消息中途暂停的情况下可靠的添加数据。 附加数据的过程应该做到：
    
    - 置位相关的 `incompat_flags` 位，
    - 设置 `append_offsets` 的第一个元素为日志文件相对于 0 的长度，
    - 然后为数据部分添加有效的任何类型的消息。
    
    这使得在将来的 Ulog 规范中在末尾添加更多的字段成为可能。 这意味着解析器必须不能假定此消息的长度是固定的。 如果消息比预期的长（当前为 40 字节），则必须忽略超过的字节。
    
    - 'F': 可以在另一个定义中作为嵌套类型记录或使用的单个 (组合) 类型的格式定义。

    struct message_format_s {
        struct message_header_s header;
        char format[header.msg_size-hdr_size];
    };
    

`format`: 具有以下格式的纯文本字符串：`message_named: field0;field1;` 可以有任意数量的字段 (至少1个) ，采用 `;` 分隔。 A field has the format: `type field_name` or `type[array_length] field_name` for arrays (only fixed size arrays are supported). `type` is one of the basic binary types or a `message_name` of another format definition (nested usage). A type can be used before it's defined. There can be arbitrary nesting but no circular dependencies.

有些字段名是特殊的：

- `timestamp`: every logged message (`message_add_logged_s`) must include a timestamp field (does not need to be the first field). Its type can be: `uint64_t` (currently the only one used), `uint32_t`, `uint16_t` or `uint8_t`. The unit is always microseconds, except for `uint8_t` it's milliseconds. A log writer must make sure to log messages often enough to be able to detect wrap-arounds and a log reader must handle wrap-arounds (and take into account dropouts). The timestamp must always be monotonic increasing for a message serie with the same `msg_id`.
- Padding: field names that start with `_padding` should not be displayed and their data must be ignored by a reader. These fields can be inserted by a writer to ensure correct alignment.
    
    If the padding field is the last field, then this field will not be logged, to avoid writing unnecessary data. This means the `message_data_s.data` will be shorter by the size of the padding. However the padding is still needed when the message is used in a nested definition.
    
    - 'I'：消息的信息。

    struct message_info_s {
        struct message_header_s header;
        uint8_t key_len;
        char key[key_len];
        char value[header.msg_size-hdr_size-1-key_len]
    };
    

`key` is a plain string, as in the format message (can also be a custom type), but consists of only a single field without ending `;`, eg. `float[3] myvalues`. `value` contains the data as described by `key`.

Note that an information message with a certain key must occur at most once in the entire log. Parsers can store information messages as a dictionary.

预定义的信息消息有：

| 键                                   | 描述                   | 示例值                |
| ----------------------------------- | -------------------- | ------------------ |
| char[value_len] sys_name          | 系统名称                 | "PX4"              |
| char[value_len] ver_hw            | 硬件版本 (主板)            | "PX4FMU_V4"        |
| char[value_len] ver_hw_subtype    | 主办子版本 (变化的)          | "V2"               |
| char[value_len] ver_sw            | 软件版本 (git 标签)        | "7f65e01"          |
| char[value_len] ver_sw_branch     | git branch           | "master"           |
| uint32_t ver_sw_release           | 软件版本 (见下文)           | 0x010401ff         |
| char[value_len] sys_os_name       | 操作系统名称               | "Linux"            |
| char[value_len] sys_os_ver        | 操作系统版本 (git 标签)      | "9f82919"          |
| uint32_t ver_os_release           | 操作系统版本 (见下文)         | 0x010401ff         |
| char[value_len] sys_toolchain     | 工具链名称                | "GNU GCC"          |
| char[value_len] sys_toolchain_ver | 工具链版本                | "6.2.1"            |
| char[value_len] sys_mcu           | 芯片名称和修订              | "STM32F42x, rev A" |
| char[value_len] sys_uuid          | 车辆的唯一标识符 (例如 MCU ID) | "392a93e32fa3"...  |
| char[value_len] replay              | 重播日志的文件名如果处于重播模式     | "log001.ulg"       |
| int32_t time_ref_utc              | UTC 时间的秒偏移量          | -3600              |

The format of `ver_sw_release` and `ver_os_release` is: 0xAABBCCTT, where AA is major, BB is minor, CC is patch and TT is the type. Type is defined as following: `>= 0`: development, `>= 64`: alpha version, `>= 128`: beta version, `>= 192`: RC version, `== 255`: release version. So for example 0x010402ff translates into the release version v1.4.2.

This message can also be used in the Data section (this is however the preferred section).

- 'M': information message multi.

    struct ulog_message_info_multiple_header_s {
        uint8_t is_continued; ///< can be used for arrays
        uint8_t key_len;
        char key[key_len];
        char value[header.msg_size-hdr_size-2-key_len]
    };
    

The same as the information message, except that there can be multiple messages with the same key (parsers store them as a list). The `is_continued` can be used for split-up messages: if set to 1, it is part of the previous message with the same key. Parsers can store all information multi messages as a 2D list, using the same order as the messages occur in the log.

- 'P': parameter message. Same format as `message_info_s`. If a parameter dynamically changes during runtime, this message can also be used in the Data section. The data type is restricted to: `int32_t`, `float`.

This section ends before the start of the first `message_add_logged_s` or `message_logging_s` message, whichever comes first.

### 数据部分

以下消息属于本部分：

- 'A': subscribe a message by name and give it an id that is used in `message_data_s`. This must come before the first corresponding `message_data_s`.

    struct message_add_logged_s {
        struct message_header_s header;
        uint8_t multi_id;
        uint16_t msg_id;
        char message_name[header.msg_size-hdr_size-3];
    };
    

`multi_id`: the same message format can have multiple instances, for example if the system has two sensors of the same type. The default and first instance must be 0. `msg_id`: unique id to match `message_data_s` data. The first use must set this to 0, then increase it. The same `msg_id` must not be used twice for different subscriptions, not even after unsubscribing. `message_name`: message name to subscribe to. Must match one of the `message_format_s` definitions.

- 'R': unsubscribe a message, to mark that it will not be logged anymore (not used currently).

    struct message_remove_logged_s {
        struct message_header_s header;
        uint16_t msg_id;
    };
    

- 'D'：包含日志数据。

    struct message_data_s {
        struct message_header_s header;
        uint16_t msg_id;
        uint8_t data[header.msg_size-hdr_size];
    };
    

`msg_id`: as defined by a `message_add_logged_s` message. `data` contains the logged binary message as defined by `message_format_s`. See above for special treatment of padding fields.

- 'L': Logged string message, i.e. printf output.

    struct message_logging_s {
        struct message_header_s header;
        uint8_t log_level;
        uint64_t timestamp;
        char message[header.msg_size-hdr_size-9]
    };
    

`timestamp`: 以微秒为单位，`log_level`: 和 Linux 内核一样。

| 名称      | 对应值 | 含义       |
| ------- | --- | -------- |
| EMERG   | '0' | 系统无法使用   |
| ALERT   | '1' | 操作必须立即执行 |
| CRIT    | '2' | 紧急情况     |
| ERR     | '3' | 错误情况     |
| WARNING | '4' | 警告情况     |
| NOTICE  | '5' | 正常但重要的情况 |
| INFO    | '6' | 信息       |
| DEBUG   | '7' | 调试级别的消息  |

- 'S': synchronization message so that a reader can recover from a corrupt message by searching for the next sync message (not used currently).

    struct message_sync_s {
        struct message_header_s header;
        uint8_t sync_magic[8];
    };
    

`sync_magic`: to be defined.

- 'O': mark a dropout (lost logging messages) of a given duration in ms. Dropouts can occur e.g. if the device is not fast enough.

    struct message_dropout_s {
        struct message_header_s header;
        uint16_t duration;
    };
    

- 'I': information message. See above.

- 'M': information message multi. See above.

- 'P': parameter message. See above.

## 解析器的要求

A valid ULog parser must fulfill the following requirements:

- Must ignore unknown messages (but it can print a warning).
- Parse future/unknown file format versions as well (but it can print a warning).
- Must refuse to parse a log which contains unknown incompatibility bits set (`incompat_flags` of `ulog_message_flag_bits_s` message), meaning the log contains breaking changes that the parser cannot handle.
- A parser must be able to correctly handle logs that end abruptly, in the middle of a message. The unfinished message should just be discarged.
- For appended data: a parser can assume the Data section exists, i.e. the offset points to a place after the Definitions section.
    
    Appended data must be treated as if it was part of the regular Data section.

## 已知的实现

- PX4 Firmware: C++ 
    - [logger module](https://github.com/PX4/Firmware/tree/master/src/modules/logger)
    - [replay module](https://github.com/PX4/Firmware/tree/master/src/modules/replay)
    - [hardfault_log module](https://github.com/PX4/Firmware/tree/master/src/systemcmds/hardfault_log): append hardfault crash data.
- [pyulog](https://github.com/PX4/pyulog): python, ULog parser library with CLI scripts.
- [FlightPlot](https://github.com/PX4/FlightPlot): Java, log plotter.
- [pyFlightAnalysis](https://github.com/Marxlp/pyFlightAnalysis): Python, log plotter and 3D visualization tool based on pyulog.
- [MAVLink](https://github.com/mavlink/mavlink): Messages for ULog streaming via MAVLink (note that appending data is not supported, at least not for cut off messages).
- [QGroundControl](https://github.com/mavlink/qgroundcontrol): C++, ULog streaming via MAVLink and minimal parsing for GeoTagging.
- [mavlink-router](https://github.com/01org/mavlink-router): C++, ULog streaming via MAVLink.
- [MAVGAnalysis](https://github.com/ecmnet/MAVGCL): Java, ULog streaming via MAVLink and parser for plotting and analysis.

## 文件格式版本历史

### 版本 2 中的改变

Addition of `ulog_message_info_multiple_header_s` and `ulog_message_flag_bits_s` messages and the ability to append data to a log. This is used to add crash data to an existing log. If data is appended to a log that is cut in the middle of a message, it cannot be parsed with version 1 parsers. Other than that forward and backward compatibility is given if parsers ignore unknown messages.