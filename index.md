class: center
name: title
count: false

# What about it?

When building software, especially in Rust, creating reliable and consistent releases is essential. In this blog, we’ll explore how to set up a robust release workflow using GitHub Actions, focusing on semantic versioning and single-click releases. It is as well highly practical for any Rust project.

By following these steps, you’ll ensure a smooth, automated pipeline for releasing software, making version management a breeze.

---

# What is a Release Workflow?

A **software release** tracks specific versions of your code so you can identify which features, bug fixes, or breaking changes your users are running. For example, when a user says they’re on version “1.2.3,” you instantly know the state of your software, including potential bugs or fixes.

A **release workflow** automates the steps required to produce valid, consistent releases of your software. It ensures every release adheres to your quality standards and is easily reproducible, reducing manual effort and potential errors.

---

# Semantic Versioning: The Key to Clear Releases

Rust, along with many other software ecosystems, uses semantic versioning to communicate expectations about releases. Semantic versioning breaks a version number into three parts: Major.Minor.Patch (e.g., `1.2.3`). Each part signifies a different level of change in your software.

# Patch Releases

- **Format:** `1.2.3 → 1.2.4` 
- **Purpose:** Fixes bugs or minor issues without introducing new features.  
- **Impact:** Users can upgrade safely without testing in most cases.  

# Minor Releases

- **Format:** `1.2.3 → 1.3.0`
- **Purpose:** Adds new features while maintaining backward compatibility.
- **Impact:** Users should test upgrades to ensure new features don’t inadvertently affect existing functionality.

# Major Releases

- **Format:** `1.2.3 → 2.0.0`
- **Purpose:** Introduces breaking changes, removes deprecated features, or redefines behaviors
- **Impact:** Users must test extensively and plan for potential disruptions.

When retiring features, it’s best practice to provide a deprecation notice in earlier versions, giving users time to adapt to the new way of doing things.

---

# Single-Click Releases in Rust Using GitHub Actions

To streamline release pipelines, I’ve designed a flexible GitHub Actions workflow that simplifies the process into a single click. This workflow is applicable to most Rust projects, and automates versioning, building, and releasing your software.

# Key Features of the Workflow

- **Versatility:** Supports major, minor, and patch releases.
- **Automation:** Automatically increments version numbers based on semantic versioning.
- **Cross-Platform Support:** Builds for multiple platforms, including Linux, macOS, and Windows.
- **Extensibility:** Can be modified to publish to [crates.io](https://crates.io/) or other targets.

# Release workflow

This release currently distributes binaries rather than crates. However, if your Rust project involves crate publishing, you can adapt the workflow by modifying the `Update Cargo.toml version and push to GitHub` step during the pre-release job.

[View Workflow](./workflow.md)

# Using the Workflow

Follow these simple steps to integrate and use the workflow:

## Step 1: Add the Workflow File
Save the [workflow file](https://github.com/Blindspot22/rustlease/blob/main/index.md) in your repository under `.github/workflows/release.yml`.

## Step 2: Create a Personal Access Token (PAT)
Create a PAT that allows the workflow to perform pushes, make tags, and perform releases for you. Store the PAT in the project secrets as `RELEASE_TOKEN`.

For reference on how to create a PAT ([LINK]()).

## Step 3: Run the Workflow

- Go to your GitHub repository and navigate to the **Actions** tab.
- Select the **Release Workflow** from the left-hand panel.
- Click **Run workflow** in the top-right corner, select the release type (major, minor, or patch), and let the automation do the rest!

---

# Why Use This Workflow?

### Benefits for Rust Projects

- **Consistency:** Ensures every release follows semantic versioning and best practices.
- **Speed:** Reduces manual intervention, allowing you to focus on development.
- **Flexibility:** Easily adaptable for various Rust workflows, whether you’re releasing binaries, libraries, or both.

---

# Final Thoughts

A well-structured release workflow is critical for maintaining a reliable software development pipeline. By using this GitHub Actions workflow, you can automate your Rust release process, adhere to semantic versioning standards, and deliver a seamless experience for your users.

Whether you’re managing a complex product or a simple Rust library, this workflow provides a solid foundation for repeatable and reliable releases.

Happy releasing!