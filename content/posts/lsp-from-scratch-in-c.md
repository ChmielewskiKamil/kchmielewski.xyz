+++
title = 'LSP From Scratch in C'
date = 2025-06-27T16:56:41+02:00
draft = true
tags = ["C", "LSP"]
author = "Kamil Chmielewski"
description = ""
+++

Intro here.

<!--more-->

Lately I've been dissatisfied with the current state of the art Language Server
for Solidity programming language. I am reviewing lots of code every day and the
fact that the Language Server is not able to provide the semantic token
highlighting to distinguish a local variable from a global one or even a
function parameter/argument is an absolute joke. I suppose dev tooling is not
easily monetizable so I don't bear a grudge as there is no incentive for anyone
to improve the status quo.

I decided to implement a Proof of Concept implementation of the Language Server
Protocol (LSP) that has only a single feature: semantic token highlighting. The
goal is to have as simple as possible program that is not bloated. For learning
purposes I will implement in in C programming language from scratch.

The [LSP
specification](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#headerPart) is
relatively straightforward and outlines what must be done. I will approach the
problem backwards. Instead of first implementing the lexer, parser and then
analyzing the Abstract Syntax Tree to provide analysis insights to the client
(code editor), I will start at the feature that I want to have and work
backwards.

The feature that I would like to have in my code editor (Neovim) is referred to
as [Semantic Token in the LSP spec](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#textDocument_semanticTokens).
Before I am able to provide it I need to somehow inform the Client (Neovim)
about the existance of my Language Server. The way you do this according to the
spec is via
[initialize](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#initialize)
"handshake". Neovim sends the Initialize Request to my Language Server and
awaits the Initialize Response. If the server successfully responds, the
connection is established. 

A question that might come to your mind is "How is my code editor able to
determine where to send the initialize request?". This is what you configure via
your plugins/extensions depending on your code editor. An extremely simple
example for doing that in [Neovim
0.11+](https://github.com/neovim/nvim-lspconfig/tree/7ad4a11cc5742774877c529fcfb2702f7caf75e4?tab=readme-ov-file#quickstart) might look like this:

```lua
-- In lsp.lua file:

-- The name that you specify here can be arbitrary.
vim.lsp.config['solbot-lsp'] = {
    cmd = { '/path/to/lsp/binary/solbot-lsp' },
    filetypes = { 'solidity' },
    root_markers = {
        '.git',
    },
}

-- configure other servers the same way
-- e.g. luals for Lua, clangd for C

vim.lsp.enable {
    'luals', -- enable configured servers
    'clangd',
    'solbot-lsp' -- solbot-lsp in particular,
}
```

Depending on your code editor the process will be different and connecting this
is not the purpose of this post. I would expect that for VSCode you will need
some boilerplate to get this up and running.

"But hey, I don't have the language server binary yet!?", don't worry, this is
what we will tackle next. You can point to a very simple hello world like
program. To visualize what the client (code editor) is sending to the language
server we can create an extremely simple program that is reading the Standard
Input (`stdin`).

The client is sending messages to the server using `stdin`. The server is
sending responses to the client through `stdout`. Any errors should go to
`stderr`.

The point that I personally was really confused about is the fact that the
Language Server Protocol is just specification and you can handle the
communication however you like. The client code editors are smart enough to be
able to determine the thing you are trying to do. The simplest way you can think
about this is that Neovim is writing his requests to a file and the language
server is reading them from that file. It could as well be a web server, where
Neovim is issuing requests to that webserver and receiving responses.

I still don't fully get this how Neovim is able to determine which transport
method to choose solely based on the `cmd` that you provide it. For example you
could [connect to the LSP via RPC](https://neovim.io/doc/user/lsp.html#lsp-rpc) `cmd = vim.lsp.rpc.connect('127.0.0.1',
4000)` and it would work the same.

One of the very first things that you will read in the spec is that each request
is divided into two parts. This includes our initialization request that we try
to implement. [There is the header part and content part](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#baseProtocol). They are separated with
the `\r\n` separator.

```txt
header part
\r\n
content part
```

The header part itself consists of header elements. These are in the format
`name:value\r\n`. Expanding on our example, the header file can look like this
with two header elements.

```txt
name:value\r\n
name:value\r\n
\r\n
content part
```

According to the current version of the specification ([`3.17`](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#headerPart)),
there are currently only two possible header fields: `Content-Length` and
`Content-Type`. Please note that it is not mandatory for both of them to be
present. As you will see in a moment we will be mostly dealing with the
`Content-Length` header element only.

```txt
Content-Length:value\r\n
Content-Type:value\r\n
\r\n
content part
```

Lets write a simple C program that will read something from the `stdin` to see
what the code editor is sending to us in the header part. The simplest idea how you might want to
implement would follow this approach:

1. Read a single line from `stdin`.
2. Print it for debugging.
3. See if the line contains exactly `\r\n` and nothing else. If that's the case
   we could stop the program since that's the end of the header part.

You can try giving it a go. Depending on how you implement debug printing
mechanism, your attempt might result in an immediate and silent failure with no
clue on what went wrong. Have a look at this program that is doing exactly that
and uses `printf(...)` for debugging.

```C
#include <stdio.h>
#include <string.h>

int main() {
  // For now we don't know how much to read. Provide an arbitrary number.
  // We will read what we can and ignore the rest of the content.
  char line_buffer[1024];
  char *separator = "\r\n";

  // The Language Server operates in an infinite loop where it reads messages
  // one by one. In our case we just want to read the header of the first
  // 'initialize' message.
  while (1) {
    // Read a single line from 'stdin'. If its EOF or an ERROR, just exit.
    if (fgets(line_buffer, sizeof(line_buffer), stdin) == NULL) {
      break;
    }

    // Debug print to 'stdout' the message that the Code Editor sent us.
    printf("Line buffer: %s", line_buffer);

    // We want to stop reading when we hit the separator between the header
    // and content part '\r\n'.
    if (strcmp(line_buffer, separator) == 0) {
      printf("Found the end of header section\n");
      break;
    }
  }

  return 0;
}
```

Calling `fgets(...)` is like telling the server "Hey, please go check the `stdin` and
see if something is there!". The server will grab a complete line (ending with
`\n`) from the `stdin`. If nothing is there it will just wait. It means that the
main server loop just stops and waits for the client to send the request. At
this point we will not be utilizing resources from the machine by looping
infinitely.

A special case that must be handled is the situation where the client exits. If
you close the code editor, the LSP should close as well. This will be denoted by
`NULL` being returned by `fgets(...)` function.

If you were to open your code editor using the code above for your LSP, you
would see... exactly nothing. For Neovim you can inspect the LSP logs (primarily errors) at `~/.local/state/nvim/lsp.log`.
Nothing shows up there. So what's going on?

Do you remember when I told you that the client is using `stdin` to communicate
with the server and the server is using `stdout` to send responses to the
client? The `printf(...)` function used in this initial implementation is
sending some stuff to the `stdout`. The code editor expects the responses to be
in a very specific format outlined by the LSP specification. If the format is
not followed, the code editor closes the connection and assumes that the server
is broken.

So how to fix this problem? We just have to use some other mechanism for
printing debug messages and leave the `stdio` for client-server communication. A
simple function that logs to a temporary log file will do the trick. On Unix
systems we can use the `tmp/` directory to store the log file. It will get
cleared on system reboot to not pollute your user space.

The `log_message(...)` function does exactly that. It opens up a file (creates
it when it does not exist) in an append mode. The message is added to the file
along the newline to make it readable. After that the file is closed.

```C
#include <stdio.h> // remember about including 'stdio.h'

void log_message(const char *message) {
  FILE *log_file = fopen("/tmp/solbot-lsp.log", "a"); // 'a' - append mode
  if (log_file != NULL) {
    fputs(message, log_file);
    fputc('\n', log_file);
    fclose(log_file);
  }
}
```

The updated `main(...)` function looks like the following (commit hash: [b2c33b4](https://github.com/ChmielewskiKamil/solbot-lsp/blob/b2c33b4245f1e41924b0cf06bd5906e34abda386/main.c)). Calls to
`printf(...)` has been replaced with `log_message(...)`. There is one extra line
at the beginning of the function to clear the log file whenever the Language
Server is launched.

```C
#include <stdio.h>
#include <string.h>

// ...

int main() {
  // 'fopen' with write mode "w" clears the file; 'fclose' immediately
  // closes it. The end result is an empty solbot-lsp.log file on each launch.
  fclose(fopen("/tmp/solbot-lsp.log", "w")); 
  log_message("--- Solbot LSP Started ---");

  char line_buffer[1024];
  char *separator = "\r\n";

  while (1) {
    if (fgets(line_buffer, sizeof(line_buffer), stdin) == NULL) {
      break;
    }

    log_message(line_buffer);

    if (strcmp(line_buffer, separator) == 0) {
      log_message("Found the end of header section");
      break;
    }
  }

  return 0;
}
```

If we run the code editor this time and inspect the newly created log file we
will be greeted with the message header. Nice.

```txt
--- Solbot LSP Started ---
Content-Length: 4272^M

^M

Found the end of header section
```

Depending on the operatins system that you are on, the resulting output might
display differently in your code editor. You can read more about [the history of
the newline control character on various operating systems on
Wikipedia](https://en.wikipedia.org/wiki/Newline#:~:text=Software%20applications%20and%20operating%20system%20representation%20of%20a%20newline%20with%20one%20or%20two%20control), 
but the takeway is that on Unix, newline is represented as `\n` and on Windows, network protocols, and our LSP it is `\r\n`. 
Since I'm on NixOS, for me the carriage return `\r` is displayed as `^M`.
It is treated as a non-printable control character. Since on Unix its not
executed (it does not bring the cursor to the beginning of the line in that
case), but it is there so it can't be ignored, it's ASCII character code is
printed: `^M`.

Why are there so many newlines, though? It's due to the fact that our
`log_message(...)` function inserts additional newline on its own. If for a
moment we remove it, the message received from the client will be easier
to interpret. Comment out the `fputc('\n', log_file);` line.

Upon opening a Solidity file in Neovim again, the log file gets populated like this:

```txt
--- Solbot LSP Started ---Content-Length: 4272

Found the end of header section
```

If for a moment we ignore the `Solbot LSP Started` part, we are left with the
message from the client. 

```txt
Content-Length: 4272'\r\n'      <-- 1st '\r\n' is encountered here
'\r\n'                          <-- 2nd '\r\n' is encountered here
Found the end of header section <-- This is our debug print. Content starts here
```

As you can see, we received a single header element: `Content-Length`. The end
of this element is denoted by the separator `\r\n` which creates the first
newline. The whole header part ends with the `\r\n` separator as well. It makes
the cursor go to the newline again, where our debug message "Found the end of
header section" is printed. From this you can see that the content section will
start after two consecutive `\r\n` separators. Great! What's next?
