This is a tiny webserver for the nodemcu platform, with an integrated online ide.

The lua backend is entirely contained in a single httpd.lua file, smaller than 4kB, still capable of handling file serving, basic authentication, websocket communication, and saving files coming in multipart form requests. The memory footprint could be smaller if some code was split in more files and loaded at every connection, but it sure would be slower, and there is still over 30kB or free heap available for the rest of application code. I havend't had any out of memory panic until now, while i've had some of those with https://github.com/marcoskirsch/nodemcu-httpserver, which is too much featurefull for my taste, so i prefer it this way.

The websocket protocol is handled thanks to the functions provided by https://github.com/creationix/nodemcu-webide, trimmed down a bit to remove support for packets bigger than 64kB. It can be easily removed from the backend in case the final project doesn't need to connect directly with the browser over tcp. To use you can pass a callback function to require("httpd")(nil,nil,nil,callback), that callback will be invoked at every websocket packet received with the decoded data as argument. To send data to websocket clients require("httpd")() will return a function capable of broadcasting a message to every connected client. It's an extremely limited API but it works for most purposes.

The ide is provided in www/edit.htm and is derived from https://github.com/me-no-dev/ESPAsyncWebServer. Once compressed with gzip of zopfli it will about 6kB, 4kB with minified css and javascript, so it will load fast. The user interface, based on Ace.js, was only tweaked to add a "restart" button, a "compile" context menu option, a REPL and serial fake terminals that only works line by line, and support for automatic compression of files with pako-deflate before saving them, depending on file extension, to speed things up.

A common app could be structured in this way:
- After getting an ip address init.lua calls a backend component called for example app.lua (or app.lc)
- app.lua sets up the microcontroller as needed for the project, defines a callback function for websocket requests, and passes it the wrapper function returned by loading httpd.lua
- the wrapper function returns a broadcast function used periodically or at certain events to send updates to clients, or something like that.
- a companion www/app.htm is created with a simple user interface that talks to the microcontroller over websockets. The wrapper function must be notified of the name of the static app.htm or it will load by default index.htm
- every modification to the dynamic lua backend or static html frontend can be done by accessing the edit.htm page, which can be password protected.

Loading the code is tricky.
The best way is to build the firmware locally and directly embed the files in the spiffs filesystem.
By using the nodemcu-builder.com service you need to populate the filesystem manually.
You can use one of https://github.com/kmpm/nodemcu-uploader or https://github.com/andidittrich/NodeMCU-Tool or any other, but my preferred way requires only a locally accessible web server (supporting plain old http, not https)

you will need to copy-paste one of these function to the lua interpreter via the serial port, then you can download any files directly to the spiffs.

#this is the size of the serial buffer (256 bytes?)
="======10========20========30========40========50========60========70========80========90=======100=======110=======120=======130=======140=======150=======160=======170=======180=======190=======200=======210=======220=======230=======240=======250===="

#this very small wget function fits the serial buffer and works everywhere (?) but the dns can be a bit finnicky (?) and must be invoked with host, 
function wget(H,U,F)C=net.createConnection(net.TCP,0)C:on("receive",function(_,R)file.write(R:match("^HTTP/1.-\r\n\r\n(.*)")or R)end)C:on("connection",function()file.open(F,"w")C:send("GET "..U.." HTTP/1.1\r\nHost:"..H.."\r\n\r\n")end)C:connect(80,H)end

#this wget function is tiny but requires the http module and doesn't work with binary data?:
wget=function(_,F)file.open(F or _:match(".*/(.-)$"),"w")http.get(_,nil,function(s,d)print(F,s,#d)file.write(d)end)end

#This larger wget is more general and works with ports other than 80
function wget(_,F)local H,U,P=_:match("(.-)(/.*)")local C=net.createConnection(net.TCP, 0)if H:match(":")then H,P=H:match("(.*):(.*)")end
file.open(F or U:match(".*/(.-)$"),"w") C:on("disconnection",file.close)C:on("receive",function(_,R)if R:match("^HTTP/1\.[0-9] 200 OK\r\n")then R=R:match("\r\n\r\n(.*)")end file.write(R)end)
C:on("connection",function()C:send("GET "..U.." HTTP/1.1\r\nHost: "..H.."\r\n\r\n")end)C:on("disconnection",function()file.close()end)net.dns.resolve(H,function(_,I)if I then C:connect(P or 80,I)end end)end

host="yourwebserver:8080"
wget(host.."/www/edit.min.htm.gz","www/edit.htm.gz")
#wget(host.."/www/index.htm","www/index.htm")
wget(host.."/init.lua","init.lua")
wget(host.."/httpd.lua","httpd.lua")
#wget(host.."/httpd.lc","httpd.lc")


modules required (cjson could be replaced by some string manipulation in the list command):
-net module
-node module
-file module
-cjson module

modules required for websocket support:
-bit module for encoding and decoding of the binary packet format
-crypto module for base64 encoder and sha1 hash, needed to initially upgrade the http connection

modules required for basic auth support:
-crypto modules for base64 encoder (could be replaced by encoder module or by hardcoded string)

notably unneeded and useless modules:
-enduser setup module

unneeded but related modules:
-mdns module
-http client module
-websocket client module

unneeded but probably useful modules:
-gpio module

substitute for the list cmd
http://witty.lan/edit?run=/local%20list={}table.foreach(file.list(),function(f,s)table.insert(list,{size=s,name=f})end)return%20cjson.encode(list)
