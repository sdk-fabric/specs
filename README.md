# Specs

This repository is part of the [SDK-Fabric](https://sdk-fabric.org/) project, an open infrastructure designed to automatically generate and maintain client SDKs across multiple programming languages.

This repository serves as the single source of truth for all client SDK specifications.

---

## How It Works

```
┌─────────────────┐      ┌─────────────────┐      ┌─────────────────────────┐
│ Edit Spec in    │ ───> │ Sync Spec to    │ ───> │ Trigger Code Gen on     │
│ ./targets/*.json│      │ TypeHub Platform│      │ Language Repositories   │
└─────────────────┘      └─────────────────┘      └─────────────────────────┘
```

1. **Edit Specification:** Modify or add a specification file under the `./targets/` directory.
2. **TypeHub Synchronization:** A GitHub Action automatically syncs changes to the [TypeHub](https://app.typehub.cloud/d/sdkfabric) platform.
3. **Automated Generation:** TypeHub detects the changes and triggers targeted workflow actions across all language repositories to regenerate the SDK code.

### Example

When an update is pushed to `./targets/openai.json`:
1. The specification is published to [TypeHub (`sdkfabric/openai`)](https://app.typehub.cloud/d/sdkfabric/openai).
2. TypeHub triggers build workflows across the individual language repositories:
    * [sdk-fabric/openai-csharp](https://github.com/sdk-fabric/openai-csharp)
    * [sdk-fabric/openai-go](https://github.com/sdk-fabric/openai-go)
    * [sdk-fabric/openai-java](https://github.com/sdk-fabric/openai-java)
    * [sdk-fabric/openai-javascript](https://github.com/sdk-fabric/openai-javascript)
    * [sdk-fabric/openai-php](https://github.com/sdk-fabric/openai-php)
    * [sdk-fabric/openai-python](https://github.com/sdk-fabric/openai-python)

### Automated Releases

Every week, TypeHub automatically creates a release tag if changes have been detected. This tag triggers a Git tag in each language repository, which automatically publishes the updated packages to their respective package managers (e.g., Packagist, npm, PyPI, NuGet, Go modules, Maven).

---

## Contributing

We welcome contributions! You can help by fixing existing specifications or adding support for new APIs.

### Adding or Modifying a Spec

1. Fork this repository and create a new branch.
2. Add your specification JSON file inside `./targets/` (e.g., `./targets/my-api.json`).
3. Ensure the spec conforms to the required specification format.
4. Submit a Pull Request.

> **Tip:** You can use AI tools or schema converters to kickstart your specification from an existing OpenAPI definition or API documentation.

---

## License

This project is licensed under the [MIT License](LICENSE).
