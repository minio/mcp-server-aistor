[![MseeP.ai Security Assessment Badge](https://mseep.net/pr/minio-mcp-server-aistor-badge.png)](https://mseep.ai/app/minio-mcp-server-aistor)

# AIStor MCP server

This is a Model Context Protocol (MCP) server for interacting with AIStor object stores.

More information about MCP can be found on its [official site](https://modelcontextprotocol.io/introduction).

More information about AIStor MCP server can be found in our [blog posts](https://blog.min.io/).

## Features

### Read operations

- List buckets
- List objects in the bucket
- Get object's metadata and tags
- Create a presigned URL with expiration for an object

### AI operations

- Describe the object's contents using the `ask-object` function

### Write operations

These operations should be enabled by using the `--allow-write` flag in the config.

- Create a bucket
- Upload an object to the bucket
- Add tags to the object

### Delete operations

These operations should be enabled by using the `--allow-delete` flag in the config.

- Delete an object
- Delete a bucket

### Admin operations

These operations should be enabled by using the `--allow-admin` flag in the config.

- Get cluster info (config, health status, available space) in human-readable form

## Configuration

### Prerequisites

1. Download and install Claude for Desktop application from [Claude](https://claude.ai/download).
   This MCP server works with other MCP-enabled clients, but let's start with Claude.

1. Make sure you have Docker or Podman installed and ready to use on your desktop/laptop.

1. Get your AIStor (or MinIO) object store credentials (`MINIO_ACCESS_KEY` and `MINIO_SECRET_KEY`) and its endpoint location.
   If you don't have an AIStor or MinIO server available, feel free to use the values for [MinIO Playground server](https://min.io/docs/minio/linux/administration/minio-console.html#logging-in).

### Configuration file

Open the Claude for Desktop configuration file with your favorite text editor.
To find the config file in your system follow the [Quickstart instructions](https://modelcontextprotocol.io/quickstart/user) from the Model Context Protocol site.
Its default location is:

- macOS: `~/Library/Application Support/Claude/claude_desktop_config.json`
- Windows: `%APPDATA%\Claude\claude_desktop_config.json`

Add the following lines to the config file, replacing the credentials, endpoint, and `YOUR_USERNAME` with their actual values.
This example uses Podman, but if you use Docker replace `podman` in the `command:` field with `docker`.
If you have other MCP servers configured just insert the AIStor config in the list.

Be careful when adding the `--allow-*` flags. By default without any flags the server will be running in read-only mode.
Add `--allow-write` and `--allow-delete` flags to this configuration only when you need them.
The `--allow-admin` flag enables admin functions that collect information about the cluster.

```json
{
  "mcpServers": {
    "aistor": {
      "command": "podman",
      "args": [
        "run",
        "-i",
        "--rm",
        "-v",
        "/Users/YOUR_USERNAME/Downloads:/Downloads",
        "-e",
        "MINIO_ENDPOINT=REPLACE_WITH_ENDPOINT",
        "-e",
        "MINIO_ACCESS_KEY=REPLACE_WITH_ACCESS_KEY",
        "-e",
        "MINIO_SECRET_KEY=REPLACE_WITH_SECRET_KEY",
        "-e",
        "MINIO_USE_SSL=true",
        "quay.io/minio/aistor/mcp-server-aistor:latest",
        "--allowed-directories",
        "/Downloads",
        "--allow-write",
        "--allow-delete",
        "--allow-admin"
      ]
    }
  }
}
```

By default, the number of objects or buckets that the server lists and send to LLM is limited by 1000 objects.
This is done to avoid hitting the limit of the model's context window in tokens.
If your model allows wider context window you can increase this number by adding the `--max-keys` parameter to the configuration file.
Keep in mind that each object record contains several dozen of tokens.

## Usage

Here are some examples of what you can do:

### Basic Operations

- "List all buckets on my AIStor server"
- "List the contents of bucket test"

### File Operations

- "Download the file example.pdf to my Downloads directory"
- "Create a bucket my-bucket"
- "Upload document.pdf from Downloads to bucket my-bucket"

### Metadata Operations

- "Get metadata of file.pdf in bucket my-bucket"
- "List all tags for object config.json"

### Admin Operations

- "Show server health status"
- "Get storage usage statistics"

Each command will return human-readable responses with relevant information.

## Available tools

Usually the Large Language Model (LLM) will be able to figure out which tool to use for your request.
Sometimes, you may want to tell the model explicitly which tool to use.
For such cases and also to let you know what else is possible with this MCP server, here is the list of tools available in this version.

You can always get this list by clicking the hammer icon in your Claude for Desktop application.

`ask-object`
: Ask questions about an object's content using AI, supporting various file formats and returning contextual answers

`copy-object`
: Copy an object from one bucket to another while preserving metadata and optionally modifying properties. You can also specify a version ID to copy a specific version of the object.

`create-bucket`
: Create a new bucket with specified configurations and optional versioning settings

`delete-bucket`
: Delete a bucket and optionally force removal of all contained objects

`delete-object`
: Delete a specific object or version from a bucket, with optional soft delete support. You can also specify a version ID to delete a specific version of the object.

`download-object`
: Download an object from a specified bucket to the local filesystem, preserving metadata. You can also specify a version ID to download a specific version of the object.

`get-admin-info`
: Get comprehensive technical information about the AIStor object store, including status, performance metrics, and configuration

`get-bucket-lifecycle`
: Get the lifecycle also known as lifecycle rules also known as ILM configuration of a specified bucket

`get-bucket-replication`
: Get the replication configuration of a specified bucket

`get-bucket-tags`
: Get the tags of a specified bucket

`get-bucket-versioning`
: Get the versioning status and configuration of a specified bucket

`get-data-usage-info`
: Get data usage information for the AIStor object storeincluding total data stored, number of objects, and usage by each bucket

`get-object-metadata`
: Get detailed metadata of an object including content type, size, custom headers, and system properties

`get-object-presigned-url`
: Get a presigned URL for an object in a bucket, with an optional expiration time. Default is 7 days.

`get-object-tags`
: Get all tags associated with a specific object in a bucket

`get-object-versions`
: Get all versions of an object in a bucket

`list-allowed-directories`
: List all directories that are permitted for operations with the server

`list-bucket-contents`
: List all objects in a specified bucket, including their sizes and last modified dates. You can also specify a prefix to filter the objects and whether to list versions.

`list-buckets`
: List all buckets in the AIStor object store with their basic information

`list-local-files`
: List all files and directories in a specified local directory path with their attributes

`move-object`
: Move an object between buckets by copying to destination and removing from source. You can also specify a version ID to move a specific version of the object.

`set-bucket-tags`
: Set the tags for a specified bucket

`set-bucket-versioning`
: Configure versioning settings for a bucket with administrative privileges

`set-object-tags`
: Set or update tags for an existing object in a bucket, supporting multiple key-value pairs. You can also specify a version ID to set tags for a specific version of the object.

`upload-object`
: Upload a file from local filesystem to a specified bucket, with support for metadata and tags
