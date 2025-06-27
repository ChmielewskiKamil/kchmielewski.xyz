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

[Commit hash reference for this version of the code on
GitHub](https://github.com/ChmielewskiKamil/solbot-lsp/blob/696dac9d308109c3e65b867ecda5cc241a0e246a/main.c).

```C
#include <stdbool.h>
#include <stdio.h>
#include <string.h>

void log_message(const char *message) {
  FILE *log_file = fopen("/tmp/solbot-lsp.log", "a"); // append mode
  if (log_file != NULL) {
    fputs(message, log_file);
    fputc('\n', log_file);
    fclose(log_file);
  }
}

int main() {
  fclose(fopen("/tmp/solbot-lsp.log", "w")); // clear the content each time
  log_message("--- LSP Server Started ---");

  while (1) {
    char line_buffer[1024];
    bool readHeader = true;
    char *separator = "\r\n";
    while (readHeader) {
      fgets(line_buffer, sizeof(line_buffer), stdin);
      log_message(line_buffer);
      if (strcmp(line_buffer, separator) == 0) {
          log_message("Found the end of header section");
          readHeader = false;
      }
    }
    return 0;
  }

  return 0;
}
```
