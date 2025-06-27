+++
title = 'Lsp From Scratch in C'
date = 2025-06-27T16:56:41+02:00
draft = true
tags = ["C", "LSP"]
author = "Kamil Chmielewski"
description = ""
+++

Intro here.

<!--more-->

Content here.

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
