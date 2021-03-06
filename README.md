# Beej's CGI C Library

_Entirely obsolete and probably horrible, I wrote this in the 90s to make it
easier to put together CGI scripts. It got some traction within my university,
but nowhere else, particularly. I include it here because I'm all nostalgic._

## Contents

* [Overview](#overview)
* [Usage](#usage)
  * [Compiling the library into your programs](compiling-the-library-into-your-programs)
  * [A POST method example](#a-post-method-example)
  * [Using the GET and ISINDEX methods](#using-the-get-and-isindex-methods)
  * [Using ISMAP](#using-ismap)
* [Security considerations](#security-considerations)
* [Quick reference of functions](#quick-reference-of-functions)
* [References](#references)


## Overview

Beej's CGI library (BCGI) is a library of C functions to handle WWW forms and
the Common Gateway Interface [1].  This library will allow a C program to handle
both GET and POST methods of form information transmission , ISINDEX, and click
maps with the ISMAP tag.  Some primitive functions are included for region
detection within a clickable map, as well.


## Usage

### Compiling the library into your programs

Using the library to handle form information is somewhat simple. When you
compile a program, you want to compile in the BCGI library, and link it to the
math library (`-lm` on Unix, usually) because BGCI makes a call to `sqrt()` for
the `distance()` function.

### A POST method example

Let's go by example, and construct a simple form that handles POST method data.

Here is a HTML document that handles the form.  Construction of form info is
available on the Web [2] along with sample CGI source [3], and is beyond the
scope of this document.

```html
<form METHOD=POST ACTION="monsearch.cgi">
    Enter the monster's character <input NAME="mchar" SIZE=2><br>
    Enter your player's level <input NAME="plev" SIZE=3><br>
    <input TYPE=submit value="Gimme the info"> or <input TYPE=reset>
</form>
```

This simple HTML page will call up a script called monsearch.cgi.
This is simply the executable from a C program that uses BCGI.  Here
is the C program source:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include "bcgi.h"

#define MAXCOMMANDLINE 255

main()
{
    char *p, *buf, commandline[MAXCOMMANDLINE], mchar;
    char tmp[200];
    int plev,i=0;
    char *arg[12];

    printf("Content-Type: text/html\n\n");

    if (strcmp(get_request_method(), "POST")) {
        printf("monsearch: REQUEST_METHOD must be POST!\n");
        exit(1);
    }

    printf("<html><head><title>Monster memory search results</title>");
    printf("<head>\n");
    printf("<body><h1>Monster memory search results</h1>\n<hr>\n<pre>\n");

    if ((buf=malloc(get_content_length()+1)) == NULL) {
        printf("Error: cannot allocate space for buffer<p></body></html>");
        exit(1);
    }

    strcpy(commandline, "/user/s2/beej/public_html/moria/mon -len 64");

    get_pdata(buf, get_content_length()+1);

    p = getvar(buf, "mchar");

    if (p != NULL && *p != '\0') {
        strcpy(tmp,p);
        escape(tmp);
        sprintf(commandline+strlen(commandline), " -mchar %s",tmp);
    }

    p = getvar(buf, "plev");
    if (p != NULL && *p != '\0') {
        strcpy(tmp,p);
        escape(tmp);
        sprintf(commandline+strlen(commandline), " -plev %s", p);
    }

    fflush(stdout);
    system(commandline);
    fflush(stdout);

    printf("</pre><hr></body></html>");
}
```

As with the POST method, the first output is that of the header (in this case
only `Content-Type:`) followed by the requested document information.

The first step this program takes is to make sure that the `REQUEST_METHOD`
environment variable is set to `POST`.  It accomplishes this by calling the
function get_request_method().  The purpose behind this step is for "CGI
compliance" [4].  Leave it out if you're lazy. ;-)

Next, a buffer is allocated to hold all of the POST data that is passed through
stdin.  The POST data is read into the buffer with a call to `get_pdata()`.
Note that the buffer must be large enough to accommodate all of the POST data,
followed by a terminating null character.  In the initial call to `malloc`, the
size of the post data is obtained through a call to `get_content_length()`.

Next, we search the buffer for the variable `mchar`.  (Remember this is the name
of the variable that we used in the `NAME=` field in the HTML form document.)
The value returned is a pointer to that null terminated data.  (The maximum
length of this data is defined in the library header file.  Note that if you
change this value, you must recompile the library.)

A call is then made to `escape()`.  This function prepends a backslash `\`
character to any characters that may have special meaning to a shell.  Since we
are using user entered options to construct a command line for `system()`, there
could be a security risk involved if we do not call `escape()`.  See [Security
Considerations](#security-considerations) below.

The process is then repeated with `plev`.  Note that `getvar()` will return
`NULL` if the variable is not found, but will return a pointer to an empty
string if the user didn't enter any data there.

Finally, the actual call to `system()` is padded with `fflush()` calls to sync
the output.

### Using the GET and ISINDEX methods

For method `GET` and `ISINDEX`, the process is similar, except no call to
`get_pdata()` is made (obviously).  The data can be retrieved with
`get_query_string()`, or through the command line options (if you so desire.)
You can then call `getvar()` with the results of `get_query_string()` to
retrieve actual variable information.

### Using ISMAP

If you're using using the `ISMAP` tag option, the mouse coordinates of the click
can be retrieved from the command line, or from `get_query_string()`.  The
coordinates will be in the form `X,Y`.


## Security considerations

There is a document on the Web that deals security considerations [5] and this
should be read before exposing your form handler to the web. If you are
careless, it might be possible for people to execute commands on your local
machine.  The `escape()` function is one of those included for your protection.
An ounce of prevention...


## Quick reference of functions

```c
int distance(int x1, int y1, int x2, int y2)
```

Returns the distance between two points (x1,y1) and (x2,y2). Useful for
calculating distance between a click point and another point.

```c
char *escape(char *s)
```

Inserts a leading `\` before any characters that might be useful to the shell.
This is included to help cover possible security risks when building a command
line with user's input.  The string s must have enough room to hold the
additional backslashes. Finally, escape returns a pointer to the escaped string.

```c
char *getvar(char *buf, char *var)
```

Searches a buffer created with `get_pdata()` for a variable with the name
pointed to by var.  `getvar()` returns a pointer to a null terminated string
containing the data.  The maximum length of this string is defined in the BCGI
header file. Subsequent calls to `getvar()` with the var parameter set to `NULL`
cause the buffer to be searched for additional occurances of the variable
initially specified.

```c
int get_content_length(void)
```

Returns as in integer the value of the `CONTENT_LENGTH` environment variable.
Useful in determining the size of buffer needed for a `get_pdata()` call.  This
function returns `NULL` on error.

```c
char *get_content_type(void)
```

Returns a pointer to the environment variable `CONTENT_TYPE`.  This function
returns `NULL` on error.

```c
char *get_gateway_interface(void)
```

Returns a pointer to the environment variable `GATEWAY_INTERFACE`. This function
returns `NULL` on error.

```c
char *get_http_user_agent(void)
```

Returns a pointer to the environment variable `HTTP_USER_AGENT`. This function
returns `NULL` on error.

```c
char *get_http_accept(void)
```

Returns a pointer to the environment variable `HTTP_ACCEPT`.  This function
returns `NULL` on error.

```c
void get_pdata(char *buf, int maxsize)
```

Fills a buffer of size `maxsize` with POST method data read from `stdin`.  If
the buffer is not large enough to hold the data AND a null terminating
character, the input data is cut short, and a null character is placed at the
end of the string.

```c
char *get_query_string(void)
```

Returns a pointer to the environment variable QUERY_STRING.  This
value can be used for method GET, or the tag ISINDEX, or the tag
option ISMAP.  This function returns NULL on error.

```c
char *get_remote_addr(void)
```

Returns a pointer to the environment variable `REMOTE_ADDR`.  This is the IP
address of the client that is accessing your document. This function returns
`NULL` on error.

```c
char *get_remote_host(void)
```

Returns a pointer to the environment variable `REMOTE_HOST`.  This is the host
name of the client that is accessing your document. This function returns `NULL`
on error.

```c
char *get_request_method(void)
```

Returns a pointer to the environment variable `REQUEST_METHOD`. This function
returns `NULL` on error.

```c
char *get_script_name(void)
```

Returns a pointer to the environment variable `SCRIPT_NAME`.  This function
returns `NULL` on error.

```c
char *get_server_name(void)
```

Returns a pointer to the environment variable `SERVER_NAME`.  This function
returns `NULL` on error.

```c
unsigned short get_server_port(void)
```

Returns the port that your server is listining on or `0` if an error occurs.

```c
char *get_server_protocol(void)
```

Returns a pointer to the environment variable `SERVER_PROTOCOL`. This function
returns `NULL` on error.

```c
char *get_server_software(void)
```

Returns a pointer to the environment variable `SERVER_SOFTWARE`. This function
returns `NULL` on error.

```c
int nearest(int x, int y, int point[], int numpoints)
```

Returns the index of the coordinate pair in the point array, with the number of
points in the array defined by numpoints, that is nearest to the point
(`x`,`y`).  This function returns `-1` if numpoints is `0` or any other error
occurs.  If there is a tie in distance, the first point in the point array will
be matched.

```c
void un_url(char *s)
```

Collapses a string that is encoded in URL form into a normal string.

```c
int xyinrect(int x, int y, int x1, int y1, int x2, int y2)
```

Returns true if the point (`x`,`y`) is inside the rectangle with upper left
coordinate (`x1`,`y1`) and lower right coordinate (`x2`,`y2`), inclusive.

```c
int xyincircle(int x, int y, int cx, int cy, int r);
```

Returns true if the point (`x`,`y`) is inside or on the circle with center
(`cx`,`cy`) and radius `r`.


## References

1. The Common Gateway Interface 
        http://hoohoo.ncsa.uiuc.edu/cgi/overview.html

2. FORM information
        http://www.ncsa.uiuc.edu/SDG/Software/Mosaic/Docs/fill-out-forms/overview.html

3. CGI sample source
        ftp://ftp.ncsa.uiuc.edu/Web/httpd/Unix/ncsa_httpd/cgi/

4. CGI compliance
        http://hoohoo.ncsa.uiuc.edu/cgi/from-htbin.html

5. Security considerations
        http://hoohoo.ncsa.uiuc.edu/cgi/security.html

