# Design Document: File permissions

## Problem statement
Currently private keys are treated as normal files, resulting in files which are
world-readable ([issue 29][1]). Private keyfiles and account keys (hereafter:
secrets) should be sufficiently protected against unauthorized access by using
an appropriate file mode.

## Goals
 - Provide a secure configuration out-of-the-box where the world cannot access
   secrets.
 - Allow permissions to be overridden in case keyfiles need more relaxed or
   restricted permissions.
 - Support different operation models:
   - The simp\_le and keyfile users are the same.
   - The simp\_le and keyfile users are different unprivileged users.
   - The simp\_le user is unprivileged, the keyfile user is root.

## Non-Goals
Certificates do not need finer permissions, these are public anyway (available
in Certificate Transparency logs).

The external plugin (`external.sh`) is a blackbox to simp\_le and and storage
issues are a concern for the external script. Therefore it is not covered
further in this document.

Only the file mode is controlled, not the file owner or group. If simp\_le were
to change the latter two attributes, it would require more privileges (which
should be discouraged).

## Design
To keep things simple, the client controls only the file mode and not ACLs. If
that functionality is desired, an external interface (`external.sh` or a wrapper
script) can be used.

The umask setting can also be used to control the file mode, but that also
affects certificate files and the Simple HTTP validation method and is
preferably not touched.

The IOPlugins that interact with secrets are:

 - `account_key.json` - account key.
 - `key.der`, `key.pem` - keyfile.
 - `full.der`, full.pem` - private keyfile, certs and chains.

The strictest possible file mode is 0400, the broadest (while considering the
goal) one is 0660. The proposed default file mode is 0600 for these reasons:

 - File mode 0400 makes it not possible for the simp\_le client to overwrite the
   keyfile as needed.
 - Some servers refuse keys that are group-accessible: [Postgres][2].

The above proposal works for the cases where the simp\_le and keyfile users are
the same or when the keyfile user is root (Apache, nginx, Dovecot, ...). Note
that although many of these services run as unprivileged user, they typically
start as root such they can bind to a privileged port and can read private key
files.

When keyfiles need to be accessible by other unprivileged users, it needs
an alternative. Some ideas:

 - Let a wrapper script create an empty key file first which is owned by the
   other user and writable by the simp\_le user (for example, using ACLs or
   group membership in the wrapper script).
 - Add a `simp\_le` option that overrides the file mode for keyfiles. Note that
   the account key is assumed private to simp\_le and not affected by this
   option (i.e. it will still have 0600 as file mode).
 - Reject this use case, pointing to the `external.sh`.

For simplicity of implementation, the first option can be chosen.

## Testing
A test must be added to check the following properties:

 - Newly created secrets must not be accessible by the world (by default).
 - Newly created keyfiles must have the requested file mode (when overridden).
 - Permissions of existing secrets should not be modified.

These tests are preferably done against the externally visible names
(`account_key.json`, etc.) as that is the public interface.

 [1]: https://github.com/kuba/simp_le/issues/29
 [2]: http://www.postgresql.org/docs/current/static/ssl-tcp.html
