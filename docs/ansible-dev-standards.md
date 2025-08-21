### Coding Standards

1. Do not hardcode any input values directly in the playbook.
2. All external input should be coming in AWX directly for higher environment usages, and for lower environments like local testing you can define input in the `vars` directory and keep separate files like `vars/common.yaml`. This helps avoid playbooks with literal values of variables, key-value pairs, or file references.

---

### Project Structure

3. Every repository must contain exactly one collection.
4. Each collection should include roles.

   * Each role should set up one large logical unit of software packages or configuration management.
5. Include all files required for Galaxy, such as `galaxy.yml`, `meta/runtime.yml`, example playbooks, and other YAML files needed for AWX, the collection, and roles.

---

### Testing

6. The root of your codebase should have a `tests` directory containing tests that can be run locally and also used in AWX.
7. Implement Molecule tests and ensure they work end-to-end for all roles.
8. Molecule tests should cover necessary aspects from a code-quality perspective, such as logical code flow with handlers, conditional execution (`when`, etc.), and proper idempotency checks wherever required.

---

### Environment Setup

9. Use a `venv`-based environment for local testing and development. For this, create a `requirements.txt` file at the root of the repository.
10. The root of the project should have a `requirements.yml` file, which AWX uses for project code refresh and environment container builds.
11. Provide shell scripts at the root of the project—one for local environment setup and another for local testing—so that other developers can use them immediately.

---

### CI/CD

12. The root of your codebase should include a well-defined GitHub workflow that builds the collection and pushes it to the public Ansible Galaxy.

---

### Documentation

13. Ensure that the project has proper and complete README files.

---