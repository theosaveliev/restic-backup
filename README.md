## General description
### Backups done right!

Or at least, that’s what the restic documentation claims.

I like the idea of using disposable hardware - or even disposable software systems - and treating failure as a normal behavior of the system, not an exceptional case.

While reading a spec of the Chord algorithm, I came across a thesis that resonated with me: we don’t need reliable storage components. Instead, we can build reliability by ensuring that data replicates slightly faster than it is destroyed.

With that in mind, let’s design a backup system for a personal laptop.

### Overview

![General overview](https://github.com/theosaveliev/restic-backup/raw/main/diagrams/backup_overview.png)

### Key points
- **No central system:**\
  Each Backup Repository is self-sufficient and operates independently.
- **Full dataset everywhere:**\
  All Backup Repositories store a complete copy of the dataset.
- **Multiple independent cloud locations:**\
  In this example, Google Drive and Drime are used. Both claim reliable storage, but real-world risks still exist - network outages, data center fires, or account lockouts. Restic uses rclone as a backend, which enables integration with many providers (67 [supported](https://rclone.org/#providers) out of the box).
- **Multiple independent offline disks:**\
  Two separate offline disks are used, so a disk can fail together with its controller or USB interface.
- **Client-side encryption:**\
  Data is encrypted before it leaves the laptop.
- **Data compression:**\
  Stored data is compressed using zstd.
- **Data deduplication:**\
  Saves disk space and network traffic by avoiding duplicate data.
- **Data integrity:**\
  Restic uses SHA-256 hashes for addressing data (content-addressable storage).
- **No monolithic containers:**\
  When a small file changes, restic does not re-upload the entire backup.
- **Packed data storage:**\
  Small data chunks are packed together, improving TCP throughput by reducing file count.
- **Consistent file naming:**\
  Avoids problems with special characters and long file names in the repository.
- **Works in userland:**\
  No privilege escalation or kernel modules are required, unlike solutions such as ZFS.
- **Open-source and cross-platform.**
- **(not implemented) Diff functionality:** https://github.com/restic/restic/issues/1620 https://github.com/restic/restic/issues/5663

## Implementation
### User interface
At a high level, the implementation is organized as follows:
- Files to backup are listed in a text file.
- Each repository has its own connection string stored in a corresponding .env file.
- Common operations are exposed as small shell commands, for example: `./restic-backup <env>`

### Environment file

Example .env file:

```
RESTIC_REPOSITORY=rclone:google:backup
RESTIC_PASSWORD_COMMAND="soda kdf laptop.passw -e base94"
RESTIC_FILES_FROM_FILE=$HOME/.restic/path.list
```

- `RESTIC_REPOSITORY` maps directly to the `--repo` argument of the `restic` command.
- `RESTIC_PASSWORD_COMMAND` is a shell command to obtain the repository password.
- `RESTIC_FILES_FROM_FILE` is a newline-separated list of filesystem paths to be backed up.

### Path list

Example path list:
```
# keys
/home/doe/.ssh/id_ed25519

# dev
/home/doe/projects

# configs
/home/doe/.ssh/config
/home/doe/.gitconfig
/home/doe/.vim

# documents
/home/doe/Documents
```

Empty lines and lines starting with # are ignored. The list is passed to the `--files-from` argument in restic.

### Available commands

The following command scripts are provided:
- `restic-backup <env>` - performs a backup to the specified destination.
- `restic-check <env>` - checks the repository for errors.
- `restic-check <env> --read-data` - performs a full read check, simulating a restore by reading all files from the backup.
- `restic-diff <env>` - shows the difference between the two most recent snapshots.

### Common sense

Let's define _a backup_ as a copy of the data in the files listed in the Path list.

Backup destinations have different space and performance limits. It’s tempting to include large files on fast local SSDs and exclude them from slower cloud storage, but that requires remembering which files live where. That creates a problem for mental simulations, where we place the system under different circumstances to find design flaws. Therefore, _a backup_ is singular: all destinations are sourced from the same data origin; there are no variations in what is backed up.

Ok, we’ve defined a _backup_. Now let’s define _sane_.

A _sane backup_ is a backup that doesn’t give you a heart attack when compromised.

On a personal laptop, it’s difficult to achieve perfect sanity, so some risks are acceptable. To manage this, I use a standard Information Classifier when reviewing the Path list. It’s fine to include everything up to the Confidential, with two assumptions:
- SSH private keys are passphrase-protected, so they are no longer considered SC.
- Connection strings in source code (/home/doe/projects) are an accepted risk when leaked.

If that’s unacceptable, the alternatives are limited:
- Exclude source code from the backup (not an option here, since that’s a major reason for backing up).
- Use a separate laptop for sensitive projects.

Since reviewing the Path list is the only use-case for the Information Classifier, I replaced a proper security guard with file manager tags:

![Finder tags](https://github.com/theosaveliev/restic-backup/raw/main/screenshots/tags.png)

### Key management
￼
![Init keys flowchart](https://github.com/theosaveliev/restic-backup/raw/main/diagrams/restic_init_keys.png)

The flow is executed during `restic init`, when the repository Access key is created. At this point, the Master key used for data encryption is generated.

The Master key is immutable and never changes over the lifetime of the repository. If the Master key is ever compromised, it cannot be revoked or rotated. In that case, the only recovery option is to remove the repository and create a new one with a fresh key.

This makes key protection critical: repository security depends entirely on keeping the Master key secret.

The good news is that, since repositories are initialized independently, each of them has its own unique Master key.

At the same time, backup restoration often happens under less-than-ideal conditions. For example, I might have just bought a new laptop and not yet logged in to any of my cloud accounts, or my usual password managers and keystores might be unavailable.

To simplify recovery I decided to use a single password for all repositories. This means the repository password is bound to the data origin rather than to the storage destination. This approach is acceptable as long as the password is strong and never reused elsewhere.

To generate such a password, I used the [soda KDF feature](https://github.com/theosaveliev/rtty-soda?tab=readme-ov-file#key-derivation), deriving it from a text source such as a book:

```
% echo "We played to an audience of five, but that’s only if you include the bartender's dog" | soda kdf - -e base94
PW=Y+6`_Q2+m6EL*@uDP,?R.b9?D-2Me]=dD5EV
```

## Q & A
### Why shell?

The short answer is: this problem _is_ shell.

The scripts do not implement business logic, data structures, or algorithms. They call external programs (`restic`, `jq`), pass arguments, and pipe commands together. This is exactly what shell scripting is designed for.

### What’s going on with the commit history?

This project is a specification, not a codebase.

GitHub is used for documentation, similar to Confluence. The history is not curated and is not meant for archaeology. If you’re looking for meaning in the commit log, you’re braver than Colombo.
