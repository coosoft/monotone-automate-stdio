# NAME

Monotone::AutomateStdio - Perl interface to Monotone via automate stdio

# VERSION

1.10

# SYNOPSIS

    use Monotone::AutomateStdio qw(:capabilities :severities);
    my(@manifest,
       $mtn,
       @revs);
    $mtn = Monotone::AutomateStdio->new("/home/fred/venge.mtn");
    $mtn->select(\@revs, "h:net.venge.monotone");
    $mtn->get_manifest_of(\@manifest, $revs[0])
        or die("mtn: " . $mtn->get_error_message());

# DESCRIPTION

The Monotone::AutomateStdio class gives a Perl developer access to Monotone's
automate stdio facility via an easy to use interface. All command, option and
output formats are handled internally by this class. Any structured information
returned by Monotone is parsed and returned back to the caller as lists of
records for ease of access (detailed below). One also has the option of
accessing Monotone's output as one large string should you prefer.

The mtn automate stdio subprocess is also controlled by this class. A new
subprocess is started, if necessary, when anything that requires it is
called. The subprocess is terminated on object destruction or when
$mtn->closedown() is called.

All automate commands have been implemented in this class except for the
\`stdio' command, hopefully the reason is obvious :-). Versions of Monotone that
are supported by this class range from 0.35 up to and including the latest
version (currently 1.10). If you happen to be using a newer version of Monotone
then this class will hopefully largely work but without the support for new or
changed features.

# CONSTRUCTORS

- **$mtn = Monotone::AutomateStdio->new(\[\\@options\])**

    Creates a new Monotone::AutomateStdio object, using the current workspace's
    database.

- **$mtn = Monotone::AutomateStdio->new\_from\_db($db\[, \\@options\])**

    Creates a new Monotone::AutomateStdio object, using the database named in $db.

- **$mtn = Monotone::AutomateStdio->new\_from\_service($service,
\[\\@options\])**

    Creates a new Monotone::AutomateStdio object, using the named network service
    (which is expressed as either a Monotone style URL or a host name optionally
    followed by a colon and the port number).

    (feature: MTN\_REMOTE\_CONNECTIONS)

- **$mtn = Monotone::AutomateStdio->new\_from\_ws($ws\[, \\@options\])**

    Creates a new Monotone::AutomateStdio object, using the workspace named in $ws.

Please note that Monotone::AutomateStdio->new() is simply an alias for
Monotone::AutomateStdio->new\_from\_db().

The @options parameter specifies what options are to be passed to the mtn
subprocess. The currently supported list (in vaguely the format in which it
would be passed to the constructor) is:

    ["--allow-default-confdir",
     "--allow-workspace",
     "--builtin-rcfile",
     "--clear-rcfiles",
     "--confdir"               => <Directory>,
     "--key"                   => <Key To Be Used For Signatures>,
     "--keydir"                => <Key Store Location>,
     "--no-builtin-rcfile",
     "--no-default-confdir",
     "--no-standard-rcfiles",
     "--no-workspace",
     "--norc",
     "--nostd",
     "--rcfile"                => <LUA File To Load>,
     "--root"                  => <Root Directory>,
     "--ssh-sign"              => One of "check", "no" or "yes",
     "--standard-rcfiles",
     "--use-default-key"]

# CLASS METHODS

- **Monotone::AutomateStdio->register\_db\_locked\_handler(\[$handler\[,
$client\_data\]\])**

    Registers the handler specified as a subroutine reference in $handler for
    database locked conditions. The value of $client\_data is simply passed to the
    handler and can be used by the caller to provide a context. This is both a
    class method as well as an object one. When used as a class method, the
    specified database locked handler is used for all Monotone::AutomateStdio
    objects that do not specify their own handlers. If no handler is given then the
    database locked condition handling is reset to the default behaviour.

    The handler subroutine is given two arguments, the first one is the
    Monotone::AutomateStdio object that encountered the locked database and the
    second is the value passed in as $client\_data when the hander was
    registered. If the handler returns true then the request that failed is
    retried, if false is returned (undef) then the default behaviour is performed,
    which is to treat it like any other error.

    Typically one would register a database locked handler when you are using this
    class in a long running application where it is quite possible that a user
    might do an mtn sync in the background. For example, mtn-browse uses this
    handler to display a \`Database is locked, please retry...' dialog window.

- **Monotone::AutomateStdio->register\_error\_handler($severity\[,
$handler\[, $client\_data\]\])**

    Registers the handler specified as a subroutine reference in $handler for
    errors of a certain severity as specified by $severity. $severity can be one of
    MTN\_SEVERITY\_WARNING, MTN\_SEVERITY\_ERROR or MTN\_SEVERITY\_ALL. The value of
    $client\_data is simply passed to the handler and can be used by the caller to
    provide a context. This is a class method rather than an object one as errors
    can be raised when calling a constructor. If no handler is given then the error
    handling is reset to the default behaviour for that severity level.

    The handler subroutine is given three arguments, the first one is a severity
    string that indicates the severity of the error being handled (either
    MTN\_SEVERITY\_WARNING or MTN\_SEVERITY\_ERROR), the second one is the error
    message and the third is the value passed in as $client\_data when the hander
    was registered.

    Please note:

    - 1)

        Warnings can be generated internally or whenever a message is received from an
        mtn subprocess that is flagged as being in error. For example, this occurs when
        an invalid selector is given to the $mtn->select() method. The subprocess
        does not exit under these conditions. Errors, however, are generated whenever
        this class detects abnormal behaviour such as output that is formatted in an
        unexpected manor or output appearing on the mtn subprocess's STDERR file
        descriptor. Under these circumstances it is expected that the application
        should exit or at least destroy any Monotone::AutomateStdio objects that are in
        error.

    - 2)

        Whilst warnings result in false being returned by those methods that return a
        boolean success indicator, errors always result in an exception being thrown.

    - 3)

        If the severity is MTN\_SEVERITY\_ERROR then it is expected that croak or die
        will be called by the handler, if this is not the case then this class will
        call croak() upon return. If you need to trap errors and prevent program exit
        then use an eval block to protect yourself in the calling code.

    - 4)

        If a warning handler is registered then this can be used to report problems via
        one routine rather than checking the boolean success indicator returned by most
        methods in this class.

    - 5)

        In order to get the severity constants into your namespace you need to use the
        following to load in this library.

            use Monotone::AutomateStdio qw(:severities);

- **Monotone::AutomateStdio->register\_io\_wait\_handler(\[$handler,
$timeout\[, $client\_data\]\])**

    Registers the handler specified as a subroutine reference in $handler for I/O
    wait conditions. This class will call the handler after waiting $timeout
    seconds for data to come from the mtn subprocess. The value of $client\_data is
    simply passed to the handler and can be used by the caller to provide a
    context. This is both a class method as well as an object one. When used as a
    class method, the specified I/O wait handler is used for all
    Monotone::AutomateStdio objects that do not specify their own handlers. If no
    handler is given then I/O wait handling is reset to the default behaviour.

    The handler subroutine is given two arguments, the first one is the
    Monotone::AutomateStdio object that is waiting for output from the mtn
    subprocess and the second is the value passed in as $client\_data when the
    handler was registered.

    Typically one would register an I/O wait handler when you are using this class
    in an interactive application where it may be necessary to do some periodic
    background processing whilst waiting for the mtn subprocess to do
    something. For example, mtn-browse uses this handler to process any outstanding
    Gtk2 events so that the application can keep its windows up to date.

- **Monotone::AutomateStdio->suppress\_utf8\_conversion($suppress)**

    Controls whether UTF-8 conversion should be done on the data sent to and from
    the mtn subprocess by this class. If $suppress is true then UTF-8 conversion is
    not done, otherwise it is. The default action is to do UTF-8 conversion. This
    is both a class method as well as an object one. When used as a class method,
    the setting is used for all applicable Monotone::AutomateStdio objects that do
    not specify their own setting.

- **Monotone::AutomateStdio->switch\_to\_ws\_root($switch)**

    Controls whether this class automatically switches to a workspace's root
    directory before running the mtn subprocess. If $switch is true then the switch
    to the workspace's root directory is done, otherwise it is not. The default
    action is to do the switch as this is generally safer. This setting only
    affects objects constructed from calling Monotone::AutomateStdio->new() and
    Monotone::AutomateStdio->new\_from\_db(). This is both a class method as well
    as an object one. When used as a class method, the setting is used for all
    applicable Monotone::AutomateStdio objects that do not specify their own
    setting.

# OBJECT METHODS

See http://monotone.ca/monotone.html for a complete description of the mtn
automate commands.

Methods that return data from the mtn subprocess do so via their first
argument. This argument is a reference to either a scalar or a list depending
upon whether the data returned by the method is raw data or a list of items
respectively. Methods that return lists of records, rather than a simple list
of scalar items, also provide the option of returning the data as one raw chunk
if the reference points to a scalar rather than a list. Therefore:

    $mtn->get_manifest_of(\$buffer);

would simply put the output from the \`get\_manifest\_of' command into the
variable named $buffer, whereas:

    $mtn->get_manifest_of(\@list);

would return the output as a list of records (actually anonymous hashes to be
precise). However:

    $mtn->get_file(\$buffer, $file_id);

will always need a reference to a scalar and:

    $mtn->select(\@list, $selector);

will always need a reference to a list (each item is just a string containing a
revision id rather than a record).

The one exception to the above is the $mtn->generate\_key() method, which
expects a reference to either a scalar or a hash as it only ever returns one
record's worth of information.

The remaining arguments depend upon the mtn command being used.

The following methods are provided:

- **$mtn->ancestors(\\@list, $revision\_id ...)**

    Get a list of ancestors for the specified revisions.

- **$mtn->ancestry\_difference(\\@list, $new\_revision\_id
\[, $old\_revision\_id ...\])**

    Get a list of ancestors for the specified revision, that are not also
    ancestors for the specified old revisions.

- **$mtn->branches(\\@list)**

    Get a list of branches.

- **$mtn->cert($revision\_id, $name, $value)**

    Add the specified certificate to the specified revision.

- **$mtn->certs(\\$buffer | \\@list, $revision\_id)**

    Get all the certificates for the specified revision. If \\$buffer is passed then
    the output from the command is simply placed into the variable. However if
    \\@list is passed then the output is returned as a list of anonymous hashes,
    each one containing the following fields:

        key       - The signer of the cert. Prior to version 0.45 of
                    Monotone this was in the form of typically an email
                    address. From version 0.45 onwards this is now a
                    hash. (feature: MTN_HASHED_SIGNATURES)
        signature - Signer status. Values can be one of "ok", "bad" or
                    "unknown".
        name      - The cert name.
        value     - Its value.
        trust     - Its trust status. Values can be one of "trusted" or
                    "untrusted".

- **$mtn->checkout(\\@options, $ws\_dir)**

    Create a new workspace in the directory specified by $ws\_dir. One can
    optionally specify the branch and revision if required. Please note that this
    command changes the current working directory of the mtn subprocess. If this is
    not what you want then the subprocess can simply be restarted by using the
    $mtn->closedown() method.

    The $options argument is a list of valid options, with some having
    arguments. For example, one could call this method specifying all of the
    options like this:

        $mtn->checkout(["branch"   => "net.venge.monotone",
                        "revision" => "t:monotone-0.35",
                        "move-conflicting-paths"],
                       "my-mtn-ws");

    (feature: MTN\_CHECKOUT)

- **$mtn->children(\\@list, $revision\_id)**

    Get a list of children for the specified revision.

- **$mtn->closedown()**

    If started then stop the mtn subprocess. Please note that the mtn subprocess is
    automatically stopped when the related object is destroyed. This method is
    provided so that application developers can conveniently control when the
    subprocess is active.

- **$mtn->common\_ancestors(\\@list, $revision\_id ...)**

    Get a list of revisions that are all ancestors of the specified revision(s).

- **$mtn->content\_diff(\\$buffer\[, \\@options\[, $revision\_id1,
$revision\_id2\[, $file\_name ...\]\]\])**

    Get the difference between the two specified revisions, optionally limiting the
    output by using the specified options and file restrictions. If the second
    revision id is undefined then the workspace's current revision is used. If both
    revision ids are undefined then the workspace's current and base revisions are
    used. If no file names are listed then differences in all files are reported.

    The $options argument is a list of valid options, with some having
    arguments. For example, one could call this method specifying all of the
    options like this (feature: MTN\_CONTENT\_DIFF\_EXTRA\_OPTIONS):

        $mtn->content_diff(\@result,
                           ["depth"   => 1,
                            "exclude" => "work.cc",
                            "reverse",
                            "with-header"],
                           "unix");

- **$mtn->db\_get(\\$buffer, $domain, $name)**

    Get the value of a database variable.

    (feature: MTN\_DB\_GET, obsolete: replaced by $mtn->get\_db\_variables())

- **$mtn->db\_locked\_condition\_detected()**

    Check to see if the Monotone database was locked the last time a command was
    issued.

- **$mtn->descendents(\\@list, $revision\_id ...)**

    Get a list of descendants for the specified revision(s).

- **$mtn->drop\_attribute($path\[, $key\])**

    Drop attributes from the specified file or directory, optionally limiting it to
    the specified attribute.

    (feature: MTN\_DROP\_ATTRIBUTE)

- **$mtn->drop\_db\_variables($domain\[, $name\])**

    Drop variables from the specified domain, optionally limiting it to the
    specified variable.

    (feature: MTN\_DROP\_DB\_VARIABLES)

- **$mtn->drop\_public\_key($key\_id)**

    Drop the public key from the database for the specified key id (either in the
    form of a name or hash id).

    (feature: MTN\_DROP\_PUBLIC\_KEY)

- **$mtn->erase\_ancestors(\\@list\[, $revision\_id ...\])**

    For a given list of revisions, weed out those that are ancestors to other
    revisions specified within the list.

- **$mtn->erase\_descendants(\\@list\[, $revision\_id ...\])**

    For a given list of revisions, weed out those that are descendants to other
    revisions specified within the list.

    (feature: MTN\_ERASE\_DESCENDANTS)

- **$mtn->file\_merge(\\$buffer, $left\_revision\_id, $left\_file\_name,
$right\_revision\_id, $right\_file\_name)**

    Get the result of merging two files, both of which are on separate revisions.

    (feature: MTN\_FILE\_MERGE)

- **$mtn->generate\_key(\\$buffer | \\%hash, $key\_id, $pass\_phrase)**

    Generate a new key for use within the database. If \\$buffer is passed then the
    output from the command is simply placed into the variable. However if \\%hash
    is passed then the output is returned as one anonymous hash containing the
    following fields:

        Prior to version 0.44 of Monotone:
            name              - The name of the key.
            public_hash       - The public hash code.
            private_hash      - The private hash code.
            public_locations  - A list of locations for the public hash
                                code. Values can be one of "database" or
                                "keystore".
            private_locations - A list of locations for the private hash
                                code. Values can be one of "database" or
                                "keystore".

        From version 0.44 of Monotone onwards
        (feature: MTN_COMMON_KEY_HASH):
            name              - The name of the key.
            hash              - The hash code (both public and private).
            public_locations  - A list of locations for the public hash
                                code. Values can be one of "database" or
                                "keystore".
            private_locations - A list of locations for the private hash
                                code. Values can be one of "database" or
                                "keystore".

    Also known as $mtn->genkey().

    (feature: MTN\_GENERATE\_KEY)

- **$mtn->get\_attributes(\\$buffer | \\@list, $file\_name\[,
$revision\_id\])**

    Get the attributes of the specified file under the specified revision. If the
    revision id is undefined then the current workspace revision is used. If
    \\$buffer is passed then the output from the command is simply placed into the
    variable. However if \\@list is passed then the output is returned as a list of
    anonymous hashes, each one containing the following fields:

        attribute - The name of the attribute.
        value     - The value of the attribute.
        state     - The status of the attribute. Values can be one of
                    "added", "changed", "dropped" or "unchanged".

    The ability to specify a revision id, thus being able to call this method
    outside of a workspace, was introduced in Monotone version 13.1 (feature:
    MTN\_GET\_ATTRIBUTES\_TAKING\_OPTIONS).

    Also known as $mtn->attributes().

    (feature: MTN\_GET\_ATTRIBUTES)

- **$mtn->get\_base\_revision\_id(\\$buffer)**

    Get the id of the revision upon which the workspace is based.

- **$mtn->get\_content\_changed(\\@list, $revision\_id, $file\_name)**

    Get a list of revisions in which the content was most recently changed,
    relative to the specified revision.

- **$mtn->get\_corresponding\_path(\\$buffer, $source\_revision\_id,
$file\_name, $target\_revision\_id)**

    For the specified file name in the specified source revision, return the
    corresponding file name for the specified target revision.

- **$mtn->get\_current\_revision(\\$buffer | \\@list\[, \\@options\[, $path
...\]\])**

    Get the revision information for the current revision, optionally limiting the
    output by using the specified options and file restrictions. If \\$buffer is
    passed then the output from the command is simply placed into the
    variable. However if \\@list is passed then the output is returned in exactly
    the same format as for the $mtn->get\_revision() method.

    The $options argument is a list of valid options, with some having
    arguments. For example, one could call this method specifying all of the
    options like this:

        $mtn->get_current_revision(\@result,
                                   ["depth"   => 1,
                                    "exclude" => "work.cc"],
                                   "unix");

    (feature: MTN\_GET\_CURRENT\_REVISION)

- **$mtn->get\_current\_revision\_id(\\$buffer)**

    Get the id of the revision that would be created if an unrestricted commit was
    done in the workspace.

- **$mtn->get\_db\_name()**

    Return the file name of the Monotone database as given to the constructor. If
    no such name was given then undef is returned.

- **$mtn->get\_db\_variables(\\$buffer | \\@list\[, $domain\])**

    Get the variables stored in the database, optionally limiting it to the
    specified domain. If \\$buffer is passed then the output from the command is
    simply placed into the variable. However if \\@list is passed then the output is
    returned as a list of anonymous hashes, each one containing the following
    fields:

        domain - The domain name to which the variable belongs.
        name   - The name of the variable.
        value  - The value of the variable.

    (feature: MTN\_GET\_DB\_VARIABLES)

- **$mtn->get\_error\_message()**

    Return the last error message received from the mtn subprocess. An empty string
    is returned if no error has occurred yet.

    Please note that the error message is never cleared but just overwritten.
    Therefore one can use this method to determinate the nature of an error once it
    has been discovered but not to actually detect it in the first place. Use
    either an error handler or check the return status of methods to detect error
    conditions.

- **$mtn->get\_extended\_manifest\_of(\\$buffer | \\@list, $revision\_id)**

    Get the extended manifest for the specified revision. If \\$buffer is passed
    then the output from the command is simply placed into the variable. However if
    \\@list is passed then the output is returned as a list of anonymous hashes,
    each one containing one or more of the following fields:

        type         - The type of entry. Values can be one of "file" or
                       "directory".
        name         - The name of the directory or file.
        file_id      - The id of the file. This field is only present if
                       type is set to "file".
        birth        - The id of the revision in which the node was first
                       added.
        content_mark - The id of the revision in which the contents of
                       the file was last changed.
        path_mark    - The id of the revision in which the file was last
                       renamed.
        size         - The size of the file's contents in bytes.
        attributes   - A list of attributes for the file or directory.
                       Each entry has the following fields:
                           attribute - The name of the attribute.
                           value     - The value of the attribute.
        attr_mark    - A list of attributes and when they were lasted
                       changed. Each entry has the following fields:
                           attribute   - The name of the attribute.
                           revision_id - The id of the revision in which
                                         the attributes value was last
                                         changed.
        dormant_attr - A list of attributes that have previously been
                       deleted.

    (feature: MTN\_GET\_EXTENDED\_MANIFEST\_OF)

- **$mtn->get\_file(\\$buffer, $file\_id)**

    Get the contents of the file referenced by the specified file id.

- **$mtn->get\_file\_of(\\$buffer, $file\_name\[, $revision\_id\])**

    Get the contents of the specified file under the specified revision. If the
    revision id is undefined then the current workspace revision is used.

- **$mtn->get\_file\_size(\\$buffer, $file\_id)**

    Get the size of the file referenced by the specified file id.

    (feature: MTN\_GET\_FILE\_SIZE)

- **$mtn->get\_manifest\_of(\\$buffer | \\@list\[, $revision\_id\])**

    Get the manifest for the current or specified revision. If \\$buffer is
    passed then the output from the command is simply placed into the
    variable. However if \\@list is passed then the output is returned as a list
    of anonymous hashes, each one containing the following fields:

        type       - The type of entry. Values can be one of "file" or
                     "directory".
        name       - The name of the directory or file.
        file_id    - The id of the file. This field is only present if
                     type is set to "file".
        attributes - A list of attributes for the file or directory. Each
                     entry has the following fields:
                         attribute - The name of the attribute.
                         value     - The value of the attribute.

- **$mtn->get\_option(\\$buffer, $option\_name)**

    Get the value of an option stored in a workspace's \_MTN directory.

- **$mtn->get\_pid()**

    Return the process id of the mtn subprocess spawned by this class. Zero is
    returned if no subprocess is thought to exist. Also if the subprocess should
    exit unexpectedly then this method will carry on returning its process id until
    the $mtn->closedown() method is called.

- **$mtn->get\_public\_key(\\$buffer, $key\_id)**

    Get the public key for the specified key id (either in the form of a name or
    hash id).

    (feature: MTN\_GET\_PUBLIC\_KEY)

- **$mtn->get\_revision(\\$buffer | \\@list, $revision\_id)**

    Get the revision information for the current or specified revision. If \\$buffer
    is passed then the output from the command is simply placed into the
    variable. However if \\@list is passed then the output is returned as a list of
    anonymous hashes, each one containing a variety of fields depending upon the
    type of entry:

        type - The type of entry. Values can be one of "add_dir",
               "add_file", "clear", "delete", "new_manifest",
               "old_revision", "patch", "rename" or "set".

        add_dir:
            name - The name of the directory that was added.

        add_file:
            name    - The name of the file that was added.
            file_id - The id of the file.

        clear:
            name      - The name of the file to which the attribute
                        applied.
            attribute - The name of the attribute that was cleared.

        delete:
            name - The name of the directory or file that was deleted.

        new_manifest:
            manifest_id - The id of the revision's new manifest.

        old_revision:
            revision_id - The id of the parent revision.

        patch:
            name         - The name of the file that was changed.
            from_file_id - The file's old id.
            to_file_id   - The file's new id.

        rename:
            from_name - The name of the file before the rename.
            to_name   - The name of the file after the rename.

        set:
            name      - The name of the file that had an attribute set.
            attribute - The name of the attribute that was set.
            value     - The value that the attribute was set to.

- **$mtn->get\_service\_name()**

    Return the service name of the Monotone server as given to the constructor.

    (feature: MTN\_REMOTE\_CONNECTIONS)

- **$mtn->get\_workspace\_root(\\$buffer)**

    Get the absolute path for the current workspace's root directory.

    (feature: MTN\_GET\_WORKSPACE\_ROOT)

- **$mtn->get\_ws\_path()**

    Return the the workspace's base directory as either given to the constructor or
    deduced from the current workspace. If neither condition holds true then undef
    is returned. Please note that the workspace's base directory may differ from
    that given to the constructor if the specified workspace path is actually a
    subdirectory within that workspace.

- **$mtn->graph(\\$buffer | \\@list)**

    Get a complete ancestry graph of the database. If \\$buffer is passed then the
    output from the command is simply placed into the variable. However if \\@list
    is passed then the output is returned as a list of anonymous hashes, each one
    containing the following fields:

        revision_id - The id of a revision.
        parent_ids  - A list of parent revision ids.

- **$mtn->heads(\\@list\[, $branch\_name\])**

    Get a list of revision ids that are heads on the specified branch. If no branch
    is given then the workspace's branch is used.

- **$mtn->identify(\\$buffer, $file\_name)**

    Get the file id, i.e. hash, of the specified file.

- **$mtn->ignore\_suspend\_certs($ignore)**

    Determine whether revisions with a suspend certificate are to be ignored or
    not. If $ignore is true then suspend certificates are ignored, otherwise they
    are honoured (in which case any suspended revisions and branches that only have
    suspended revisions on their heads will not be listed). The default behaviour
    is to honour suspend certificates.

    (feature: MTN\_IGNORING\_OF\_SUSPEND\_CERTS)

- **$mtn->interface\_version(\\$buffer)**

    Get the version of the mtn automate interface.

- **$mtn->inventory(\\$buffer | \\@list\[, \\@options\[, $path ...\]\])**

    Get the inventory for the current workspace, optionally limiting the output by
    using the specified options and file restrictions. If \\$buffer is passed then
    the output from the command is simply placed into the variable. However if
    \\@list is passed then the output is returned as a list of anonymous hashes,
    each one containing one or more of the following fields:

        Prior to version 0.37 of Monotone:
            status       - The three inventory status characters for the
                           file or directory.
            crossref_one - The first cross-referencing number.
            crossref_two - The second cross-referencing number.
            name         - The name of the file or directory.

        From version 0.37 of Monotone onwards
        (feature: MTN_INVENTORY_IN_IO_STANZA_FORMAT):
            path     - The name of the file or directory.
            old_type - The type of the entry in the base manifest. Values
                       can be one of "directory", "file" or "none".
            new_type - The type of the entry in the revision manifest.
                       Values can be one of "directory", "file" or
                       "none".
            fs_type  - The type of the entry on the file system. Values
                       can be one of "directory", "file" or "none".
            old_path - The old name of the file or directory if it has
                       been renamed in the revision manifest.
            new_path - The new name of the file or directory if it has
                       been renamed in the revision manifest.
            birth    - The id of the revision in which the node was first
                       added. Only from version 0.41 of Monotone
                       onwards. (feature: MTN_INVENTORY_WITH_BIRTH_ID)
            status   - A list of status flags. Values can be one of
                       "added", "dropped", "ignored", "invalid", "known",
                       "missing", "rename_source", "rename_target" or
                       "unknown".
            changes  - A list of change flags. Values can be one of
                       "attrs" or "content".

    Please note that some fields are not used by all entries, in which case they
    are not present (use Perl's exists() function to determine their presence and
    not defined()).

    The $options argument is a list of valid options, with some having
    arguments. For example, one could call this method specifying all of the
    options like this (feature: MTN\_INVENTORY\_TAKING\_OPTIONS):

        $mtn->inventory(\@result,
                        ["depth"   => 1,
                         "exclude" => "work.cc",
                         "no-corresponding-renames",
                         "no-ignored",
                         "no-unknown",
                         "no-unchanged"],
                        "unix");

- **$mtn->keys(\\$buffer | \\@list)**

    Get a list of all the keys known to mtn. If \\$buffer is passed then the output
    from the command is simply placed into the variable. However if \\@list is
    passed then the output is returned as a list of anonymous hashes, each one
    containing the following fields:

        Prior to version 0.44 of Monotone:
            name              - The name of the key.
            public_hash       - The public hash code.
            private_hash      - The private hash code.
            public_locations  - A list of locations for the public hash
                                code. Values can be one of "database" or
                                "keystore".
            private_locations - A list of locations for the private hash
                                code. Values can be one of "database" or
                                "keystore". This field is only present if
                                there is a private key.

        Version 0.44 of Monotone (feature: MTN_COMMON_KEY_HASH):
            name              - The name of the key.
            hash              - The hash code (both public and private).
            public_locations  - A list of locations for the public hash
                                code. Values can be one of "database" or
                                "keystore".
            private_locations - A list of locations for the private hash
                                code. Values can be one of "database" or
                                "keystore". This field is only present if
                                there is a private key.

        From version 0.45 of Monotone onwards
        (feature: MTN_HASHED_SIGNATURES):
            given_name        - The name of the key as given when it was
                                created.
            hash              - The hash code (both public and private).
            local_name        - The local name of the key as returned by
                                the get_local_key_name() hook.
            public_locations  - A list of locations for the public hash
                                code. Values can be one of "database" or
                                "keystore".
            private_locations - A list of locations for the private hash
                                code. Values can be one of "database" or
                                "keystore". This field is only present if
                                there is a private key.

    Please note that some fields are not used by all entries, in which case they
    are not present (use Perl's exists() function to determine their presence and
    not defined()).

- **$mtn->leaves(\\@list)**

    Get a list of leaf revisions.

- **$mtn->log(\\@list\[, \\@options\[, $file\_name\]\])**

    Get a list of revision ids that form a log history for an entire project,
    optionally limiting the output by using the specified options and file name
    restrictions.

    The $options argument is a list of valid options, with some having
    arguments. For example, one could call this method specifying all of the
    options like this:

        $mtn->log(\@list,
                  ["clear-from",
                   "clear-to",
                   "from"       => "h:",
                   "last"       => 20,
                   "next"       => 30,
                   "no-merges"
                   "to"         => "t:test-checkin"],
                  "Makefile.am");

    (feature: MTN\_LOG)

- **$mtn->lua(\\$buffer, $lua\_function\[, $argument ...\])**

    Call the specified LUA function with any required arguments.

    (feature: MTN\_LUA)

- **$mtn->packet\_for\_fdata(\\$buffer, $file\_id)**

    Get the contents of the file referenced by the specified file id in packet
    format.

- **$mtn->packet\_for\_fdelta(\\$buffer, $from\_file\_id, $to\_file\_id)**

    Get the file delta between the two files referenced by the specified file ids
    in packet format.

- **$mtn->packet\_for\_rdata(\\$buffer, $revision\_id)**

    Get the contents of the revision referenced by the specified revision id in
    packet format.

- **$mtn->packets\_for\_certs(\\$buffer, $revision\_id)**

    Get all the certs for the revision referenced by the specified revision id in
    packet format.

- **$mtn->parents(\\@list, $revision\_id)**

    Get a list of parents for the specified revision.

- **$mtn->pull(\\$buffer | \\@list\[,@options\[, $uri\]\])**

    Synchronises database changes from the specified remote server to the local
    database but not in the other direction. Other details are identical to the
    $mtn->sync() method.

    (feature: MTN\_SYNCHRONISATION)

- **$mtn->push(\\$buffer | \\@list\[,@options\[, $uri\]\])**

    Synchronises database changes from the local database to the specified remote
    server but not in the other direction. Other details are identical to the
    $mtn->sync() method.

    (feature: MTN\_SYNCHRONISATION)

- **$mtn->put\_file(\\$buffer, $base\_file\_id, \\$contents)**

    Put the specified file contents into the database, optionally basing it on the
    specified file id (this is used for delta encoding). The file id is returned.

- **$mtn->put\_public\_key($public\_key)**

    Put the specified public key data into the database.

    (feature: MTN\_PUT\_PUBLIC\_KEY)

- **$mtn->put\_revision(\\$buffer, \\$contents)**

    Put the specified revision data into the database. The revision id is
    returned. Please note that any newly created revisions have no certificates
    associated with them and so these have to be added using the $mtn->cert()
    method.

- **$mtn->read\_packets($packet\_data)**

    Decode and store the specified packet data in the database.

    (feature: MTN\_READ\_PACKETS)

- **$mtn->register\_error\_handler($severity\[, $handler
\[, $client\_data\]\])**

    Registers an error handler for the object rather than the class. For further
    details please see the description of the class method.

- **$mtn->register\_db\_locked\_handler(\[$handler\[, $client\_data\]\])**

    Registers a database locked handler for the object rather than the class. For
    further details please see the description of the class method.

- **$mtn->register\_io\_wait\_handler(\[$handler, $timeout
\[, $client\_data\]\])**

    Registers an I/O wait handler for the object rather than the class. For further
    details please see the description of the class method.

- **$mtn->register\_stream\_handle($stream\[, $handle\]\])**

    Registers the file handle specified in $handle so that it will receive data
    from the specified mtn output stream. $stream can be one of MTN\_P\_STREAM or
    MTN\_T\_STREAM. If no file handle is given then any existing file handle is
    unregistered for that stream.

    Please note:

    - 1)

        It is vitally important that if you register some sort of pipe or socket to
        receive mtn stream output, that any data sent down it is read immediately and
        independently from the code calling the method generating the output (either by
        using a thread or another process). Not doing so could easily cause a deadlock
        situation to occur where the method stops processing, waiting for the pipe or
        socket to empty before proceeding.

    - 2)

        The output streams are largely sent as received from the mtn subprocess (please
        refer to the Monotone documentation for further details on the format). The
        only difference is that since the \`l' or last message (which marks the end of a
        command's output) is only sent once by mtn, this class duplicates it onto any
        registered stream so that the reader knows when there is no more data for a
        command.

    - 3)

        In order to get the stream constants into your namespace you need to use the
        following to load in this library.

            use Monotone::AutomateStdio qw(:streams);

    (feature: MTN\_STREAM\_IO)

- **$mtn->roots(\\@list)**

    Get a list of root revisions, i.e. revisions with no parents.

- **$mtn->select(\\@list, $selector)**

    Get a list of revision ids that match the specified selector.

- **$mtn->set\_attribute($path, $key, $value)**

    Set an attribute on the specified file or directory.

    (feature: MTN\_SET\_ATTRIBUTE)

- **$mtn->set\_db\_variable($domain, $name, $value)**

    Set the value of a database variable.

    Also known as $mtn->db\_set().

    (feature: MTN\_SET\_DB\_VARIABLE)

- **$mtn->show\_conflicts(\\$buffer | \\@list\[, $branch\]\[,
$left\_revision\_id, $right\_revision\_id\])**

    Get a list of conflicts between the first two head revisions on the current
    branch, optionally one can specify both head revision ids and the name of the
    branch that they reside on. If \\$buffer is passed then the output from the
    command is simply placed into the variable. However if \\@list is passed then
    the output is returned as a list of anonymous hashes, each one containing one
    or more of the following fields:

        ancestor          - The id of the common ancestor revision for
                            both revisions in conflict.
        ancestor_file_id  - The common ancestor file id for both files in
                            conflict.
        ancestor_name     - The name of the ancestor file or directory.
        attr_name         - The name of the Monotone file or directory
                            attribute that is in conflict.
        conflict          - The nature of the conflict. Values can be one
                            of "attribute", "content",
                            "directory_loop", "duplicate_name",
                            "invalid_name", "missing_root",
                            "multiple_names", "orphaned_directory" or
                            "orphaned_file".
        left              - The id of the left hand revision that is in
                            conflict.
        left_attr_value   - The value of the attribute on the file or
                            directory in the left hand revision.
        left_file_id      - The id of the file in the left hand revision.
        left_name         - The name of the file or directory in the left
                            hand revision.
        left_type         - The type of conflict relating to the file or
                            directory in the left hand revision. Values
                            can be one of "added directory",
                            "added file", "deleted directory",
                            "pivoted root", "renamed directory" or
                            "renamed file".
        node_type         - The type of manifest node. Values can be one
                            of "file" or "directory".
        resolved_internal - Only present if the conflict can be resolved
                            internally by Monotone during the merge
                            process.
        right             - The id of the right hand revision that is in
                            conflict.
        right_attr_state  - The state of the attribute in the right hand
                            revision. Values can only be "dropped".
        right_attr_value  - The value of the attribute on the file or
                            directory in the right hand revision.
        right_file_id     - The id of the file in the right hand
                            revision.
        right_name        - The name of the file or directory in the
                            right hand revision.
        right_type        - The type of conflict relating to the file or
                            directory in the left revision. Values are as
                            documented for left_type.

    Please note that some fields are not used by all entries, in which case they
    are not present (use Perl's exists() function to determine their presence and
    not defined()).

    (feature: MTN\_SHOW\_CONFLICTS)

- **$mtn->supports($feature)**

    Determine whether a certain feature is available with the version of Monotone
    that is currently being used by this object. The list of valid features are:

        MTN_CHECKOUT
        MTN_COMMON_KEY_HASH
        MTN_CONTENT_DIFF_EXTRA_OPTIONS
        MTN_DB_GET
        MTN_DROP_ATTRIBUTE
        MTN_DROP_DB_VARIABLES
        MTN_DROP_PUBLIC_KEY
        MTN_ERASE_DESCENDANTS
        MTN_FILE_MERGE
        MTN_GENERATE_KEY
        MTN_GET_ATTRIBUTES
        MTN_GET_ATTRIBUTES_TAKING_OPTIONS
        MTN_GET_CURRENT_REVISION
        MTN_GET_DB_VARIABLES
        MTN_GET_EXTENDED_MANIFEST_OF
        MTN_GET_FILE_SIZE
        MTN_GET_PUBLIC_KEY
        MTN_GET_WORKSPACE_ROOT
        MTN_HASHED_SIGNATURES
        MTN_IGNORING_OF_SUSPEND_CERTS
        MTN_INVENTORY_IN_IO_STANZA_FORMAT
        MTN_INVENTORY_TAKING_OPTIONS
        MTN_INVENTORY_WITH_BIRTH_ID
        MTN_K_SELECTOR
        MTN_LOG
        MTN_LUA
        MTN_M_SELECTOR
        MTN_P_SELECTOR
        MTN_PUT_PUBLIC_KEY
        MTN_READ_PACKETS
        MTN_REMOTE_CONNECTIONS
        MTN_SELECTOR_FUNCTIONS
        MTN_SELECTOR_MIN_FUNCTION
        MTN_SELECTOR_NOT_FUNCTION
        MTN_SELECTOR_OR_OPERATOR
        MTN_SET_ATTRIBUTE
        MTN_SET_DB_VARIABLE
        MTN_SHOW_CONFLICTS
        MTN_STREAM_IO
        MTN_SYNCHRONISATION
        MTN_SYNCHRONISATION_WITH_OUTPUT
        MTN_U_SELECTOR
        MTN_UPDATE
        MTN_W_SELECTOR

    In order to get these constants into your namespace you need to use the
    following to load in this library.

        use Monotone::AutomateStdio qw(:capabilities);

    Please note that if you see (feature: ...) then this means that whatever is
    being discussed is only available if $mtn->supports() returns true for the
    specified feature.

- **$mtn->sync(\\$buffer | \\@list\[,@options\[, $uri\]\])**

    Synchronises database changes between the local database and the specified
    remote server. $uri specifies the name of the server, the port, the database
    and the branch pattern to synchronise to, for example
    "mtn://code.monotone.ca:8000/monotone?net.venge.monotone\*". If \\$buffer is
    passed then the output from the command is simply placed into the
    variable. However if \\@list is passed then the output is returned as a list of
    anonymous hashes, each one containing one or more of the following fields
    (feature: MTN\_SYNCHRONISATION\_WITH\_OUTPUT):

        key              - The signing key id for the current item.
        receive_cert     - The name of the certificate that has just been
                           received.
        receive_key      - The id of the key that has just been received.
        receive_revision - The id of the revision that has just been
                           received.
        revision         - The id of the revision relating to the current
                           item.
        send_cert        - The name of the certificate that has just been
                           sent.
        send_key         - The id of the key that has just been sent.
        send_revision    - The id of the revision that has just been
                           sent.
        value            - The value of the certificate.

    The $options argument is a list of valid options, with some having
    arguments. For example, one could call this method specifying all of the
    options like this:

        $mtn->sync(\$buffer,
                   ["dry-run",
                    "exclude"             => "experiments.hacks",
                    "key-to-push"         => "me@mydomain.com",
                    "max-netsync-version" => 2,
                    "min-netsync-version" => 1,
                    "set-default"],
                   "mtn://code.monotone.ca/monotone?net.venge.monotone");

    (feature: MTN\_SYNCHRONISATION)

- **$mtn->tags(\\$buffer | \\@list\[, $branch\_pattern\])**

    Get all the tags attached to revisions on branches that match the specified
    branch pattern. If no pattern is given then all branches are searched. If
    \\$buffer is passed then the output from the command is simply placed into the
    variable. However if \\@list is passed then the output is returned as a list of
    anonymous hashes, each one containing the following fields:

        tag         - The name of the tag.
        revision_id - The id of a revision that the tag is attached to.
        signer      - The signer of the tag. Prior to version 0.45 of
                      Monotone this was in the form of typically an email
                      address. From version 0.45 onwards this is now a
                      hash. (feature: MTN_HASHED_SIGNATURES)
        branches    - A list of all branches that contain this revision.

- **$mtn->toposort(\\@list\[, $revision\_id ...\])**

    Sort the specified revision ids such that the ancestors come out first.

- **$mtn->update(\[@options\])**

    Updates the current workspace to the specified revision and possible branch. If
    no options are specified then the workspace is updated to the head revision of
    the current branch.

    The $options argument is a list of valid options, with some having
    arguments. For example, one could call this method specifying all of the
    options like this:

        $mtn->update(["branch"                  => "experiments.hacks",
                      "move-conflicting-paths",
                      "revision"                => "h:"]);

    (feature: MTN\_UPDATE)

# RETURN VALUE

Except for those methods listed below, all remaining methods return a boolean
success indicator, true for success or false for failure.

The following constructors return Monotone::AutomateStdio objects:

    Monotone::AutomateStdio->new()
    Monotone::AutomateStdio->new_from_db()
    Monotone::AutomateStdio->new_from_service()
    Monotone::AutomateStdio->new_from_ws()

The following method returns true or false (but it does not indicate success or
failure):

    $mtn->db_locked_condition_detected()

The following method returns an integer:

    $mtn->get_pid()

The following methods return a string or undef:

    $mtn->get_db_name()
    $mtn->get_error_message()
    $mtn->get_ws_path()

The following methods do not return anything:

    Monotone::AutomateStdio->register_db_locked_handler()
    Monotone::AutomateStdio->register_error_handler()
    Monotone::AutomateStdio->register_io_wait_handler()
    Monotone::AutomateStdio->register_stream_handle()
    Monotone::AutomateStdio->suppress_utf8_conversion()
    $mtn->closedown()

Please note that the boolean true and false values used by this class are
defined as 1 and undef respectively.

# EXAMPLES

## Detecting Warnings And Errors

Errors cause exceptions to be thrown. Warnings cause the responsible method to
return false rather than true.

Therefore warnings would typically be detected by using code like this:

    $mtn->get_file(\$data, $file_id)
        or die('$mtn->get_file() failed with: '
               . $mtn->get_error_message());

However this can get clumsy and cluttered after a while, especially when the
calling application knows that there is very little likelihood of there being a
problem. Having exceptions being thrown for warnings as well as errors may be a
better approach. By doing:

    Monotone::AutomateStdio->register_error_handler
        (MTN_SEVERITY_ALL,
         sub
         {
             my($severity, $message) = @_;
             die($message);
         });

or more tersely:

    Monotone::AutomateStdio->register_error_handler
        (MTN_SEVERITY_ALL, sub { die($_[1]); });

at the beginning of your application will mean that all errors and warnings
detected by this class will generate an exception, thus making the checking of
a method's return status redundant.

## Silently Retrying Operations When The Database Is Locked

If the Monotone database is locked then, by default, this class will report
that condition as a warning. However it may be more convenient to ask this
class to silently retry the operation until it succeeds. This can easily be
done by using the database locked handler as follows:

    Monotone::AutomateStdio->register_db_locked_handler
        (sub { sleep(1); return 1; });

This will mean that should any database locked conditions occur then this class
will silently sleep for one second before retrying the operation.

## Dealing With Processing Lockouts And Delays

Some requests sent to the mtn subprocess can take several seconds to complete
and consequently this class will take that amount of time, plus a very small
processing overhead, to return. Whilst one can get around this by using
threads, another way is to register an I/O wait handler. For example:

    Monotone::AutomateStdio->register_io_wait_handler
        (sub { WindowManager->instance()->update_gui(); }, 1);

will instruct this class to update the user display every second whilst it is
waiting for the mtn subprocess to finish processing a request.

# NOTES

## Using This Class With Monotone Workspaces

Certain features are only available when used inside a Monotone workspace, so
consequently this class does support the use of workspaces. However certain
aspects of a workspace can affect the mtn subprocess. For example, when you run
mtn in a workspace's subdirectory then all file name arguments get translated
into paths relative to the workspace's root directory (i.e. given a path of
/home/freddy/workspace/include/system.h when one runs `mtn log system.h` in
the include directory the file name gets changed to include/system.h).

This makes perfect sense on the command line but possibly less so when using
mtn from inside a GUI application where the directory in which the application
was launched is not as relevant. This is why this class runs the mtn subprocess
in specific directories depending upon how the object is constructed. If the
object is constructed by calling Monotone::AutomateStdio->new() or
Monotone::AutomateStdio->new\_from\_db() with the name of a database, then
the mtn subprocess is run in the root directory, otherwise it is run in the
workspace's root directory. This guarantees correct behaviour.

If however you want the mtn subprocess to run in the current directory within a
workspace then simply use the Monotone::AutomateStdio->switch\_to\_ws\_root()
method to before calling Monotone::AutomateStdio->new() without specifying
a database.

Any changing of directory does not affect the caller.

## UTF-8 Handling

A Monotone database may contain UTF-8 characters. These characters would most
commonly appear inside text files but may also appear in the names of files,
branches and even in certificates. Later versions of Perl have built in support
for UTF-8 strings and can represent this sort of data quite naturally without
the developer really having to worry too much. For this reason this class
automatically converts any data sent between Perl and a mtn subprocess to and
from Perl's internal UTF-8 string format and the standard binary octet
notation.

There may be times when this built in conversion is inappropriate and so one
can simply switch it off by using the
Monotone::AutomateStdio->suppress\_utf8\_conversion() method to before
calling Monotone::AutomateStdio->new().

Please note that not everything is converted when using this class's built in
UTF-8 conversion mechanism. This is mainly for efficiency. For example, there
is no real point in converting revision or file ids as these are always
represented as forty character hexadecimal strings. Likewise packet data is not
converted as this is always formatted to use seven bit ASCII. However there is
one case where no conversion is done for reasons other than efficiency, and
that is when handling file content data. Apart from differentiating between
binary and non-binary data, Monotone just treats file content data as a
sequence of bytes. In order to decide whether a file's contents contains UTF-8
data this may involve looking for assorted patterns in the data or checking the
file name's extension, all of which being beyond the scope of this class. Also
it is a good idea to treat file content data as a simple sequence of octets
anyway. Probably the only time that one would need to worry about UTF-8 based
file content data is when an application is displaying it using something like
Gtk2 (the Gtk2 bindings for Perl uses Perl's internal UTF-8 flag to determine
whether a string needs to be handled as a UTF-8 string or as a simple sequence
of octets). In this case, once a file's contents has been determined to contain
text oriented data, one can use Perl's decode\_utf8() function to do the
conversion. For more information on Perl's handling of UTF-8 please see the
documentation for Encode.

## Process Handling

There are situations where this class does legitimately terminate the mtn
subprocess (for example when a database locked condition is detected). When
this happens the subprocess is reaped and its id is reset, i.e. the
$mtn->get\_pid() method returns 0. However if the subprocess should exit
unexpectedly whilst processing a request then an exception is raised but no
reaping or process id resetting takes place. Therefore an application using
this class may wish to have a signal handler registered for SIGCHILD signals
that can indirectly trigger a call to the $mtn->closedown() method or
destroy the object concerned in the event of an error. In order to distinguish
between legitimate terminations of the mtn subprocess and failures, simply
compare the reaped process id against that returned by the $mtn->get\_pid()
method. If there is a match then there is a problem, otherwise, as far as this
class is concerned, there is nothing to worry about.

Please note that all SIGPIPE signals are set to be ignored as soon as the first
mtn subprocess is started by this class.

## Use Of SIGALRM

In order to reliably shutdown the mtn subprocess, alarm() is used to timeout
calls to waitpid(), allowing this class to kill off any obstinate mtn
subprocesses that remain after having their STDIN, STDOUT and STDERR file
descriptors closed. Normally closing the file descriptors will cause a clean
exit, I have never known it not to, at which point any alarms are reset without
any SIGALRM signal being generated.

## General

When the output of a command from the automate stdio interface changes
dramatically, it will probably not be possible to protect Perl applications
from those changes. This class is a convenience wrapper rather than something
that will totally divorce you from the automate stdio interface. Also the
chances are you will want the new type of output anyway.

No work will be done to support versions of Monotone older than 0.35, so if you
are in that position then I strongly suggest that you upgrade to a later
version of Monotone (you will get all the new features and bug fixes, go on you
know want them :-)). Also once the automate stdio interface has remained stable
for some time, support may be dropped for older versions in order to aid
maintenance and regression testing.

The $mtn->get\_content\_changed() method is very slow in Monotone versions
0.40 and 0.41.

# SEE ALSO

http://monotone.ca

# BUGS

Your mileage may vary if this class is used with versions of Monotone older
than 0.35 (automate stdio interface version 4.3).

This class is not thread safe. If you wish to use this class in a
multi-threaded environment then you either need to use a separate object per
thread or use threads::lock to protect each method call.

# AUTHORS

Anthony Cooper with contributions and ideas from Thomas Keller. Currently
maintained by Anthony Cooper. Please report all faults and suggestions to
<support@coosoft.plus.com>.

# COPYRIGHT

Copyright (C) 2008 by Anthony Cooper <aecooper@coosoft.plus.com>.

This library is free software; you can redistribute it and/or modify it under
the terms of the GNU Lesser General Public License as published by the Free
Software Foundation; either version 3 of the License, or (at your option) any
later version.

This library is distributed in the hope that it will be useful, but WITHOUT ANY
WARRANTY; without even the implied warranty of MERCHANTABILITY or FITNESS FOR A
PARTICULAR PURPOSE. See the GNU Lesser General Public License for more details.

You should have received a copy of the GNU Lesser General Public License along
with this library; if not, write to the Free Software Foundation, Inc., 59
Temple Place - Suite 330, Boston, MA 02111-1307 USA.
