# How to Encrypt the SARIF Artifact

**This is optional.** It is running in `platform-backend` and `platform-ui`
because those repositories are public and their findings are sensitive.
Most projects do not need it.

## The problem it solves

Each pipeline publishes its findings twice, and the two channels do not have
the same access rules:

| Channel | Who can see it |
|---|---|
| Security tab (`upload-sarif`) | People with **write** access, so your team |
| Workflow artifact (`upload-artifact`) | Anyone with **read** access, which on a public repository is every signed-in GitHub user |

On a public repository the artifact is a downloadable list of exactly which
unpatched CVEs are in the running system, available to anyone with an
account. For MIP, that is a medical data platform.

## Three options

Pick one. Encryption is only the third.

- **Drop the artifact.** Delete the `upload-artifact` step. The Security tab
  still receives everything and your team loses only the downloadable copy.
  Simplest, and correct for most projects.
- **Leave it public.** Fine where the findings are not sensitive.
- **Encrypt it.** Keep the artifact, make it useless without the password.

## Setting it up

Create a repository secret named `ARTIFACT_PASSWORD`, then put this between
the scan step and the upload step:

```yaml
      - name: Encrypt SARIF report
        id: encrypt
        if: ${{ !cancelled() && hashFiles(env.OPENGREP_SARIF_OUTPUT) != '' }}
        env:
          PW: ${{ secrets.ARTIFACT_PASSWORD }}
          OUT: ${{ env.OPENGREP_SARIF_OUTPUT }}.gpg
        run: |
          if [ -z "$PW" ]; then
            echo "::warning title=SARIF not uploaded::ARTIFACT_PASSWORD unavailable (Dependabot, Renovate or fork PR). Skipping."
            rm -f -- "$OPENGREP_SARIF_OUTPUT"
            exit 0
          fi
          printf '%s' "$PW" | gpg --symmetric --batch --yes --quiet \
            --pinentry-mode loopback --passphrase-fd 0 \
            --no-symkey-cache --cipher-algo AES256 \
            --s2k-mode 3 --s2k-digest-algo SHA512 --s2k-count 65011712 \
            --output "$OUT" -- "$OPENGREP_SARIF_OUTPUT"
          rm -f -- "$OPENGREP_SARIF_OUTPUT"
          echo "encrypted=$OUT" >> "$GITHUB_OUTPUT"

      - name: Upload encrypted SARIF artifact
        if: ${{ !cancelled() && steps.encrypt.outputs.encrypted != '' }}
        uses: actions/upload-artifact@v7
        with:
          name: sast-scan-sarif-report
          path: ${{ steps.encrypt.outputs.encrypted }}
          retention-days: 30
          if-no-files-found: error
```

Swap `OPENGREP_SARIF_OUTPUT` for the report variable of the pipeline you are
editing: `SCA_MERGED_SARIF_OUTPUT` for SCA, `MERGED_SARIF_OUTPUT` for
Container Scanning.

## Points worth knowing

- **The plaintext SARIF is deleted in both branches.** Whether encryption
  runs or is skipped, the unencrypted file is removed before the upload step
  can see it.
- **Runs without the secret skip the upload.** Dependabot, Renovate and
  pull requests from forks do not receive repository secrets. Rather than
  failing the job or uploading in the clear, the step logs a workflow
  warning and uploads nothing.
- **The Security tab copy stays unencrypted, deliberately.** It is already
  restricted to people with write access, so there is nothing there to
  protect it from. Encrypting the artifact closes the one channel that was
  open.
- **It is not in the blueprint.** `reusable-ci-pipelines` documents this as
  an option rather than shipping it, because most adopters do not need it.

## Reading a report

```bash
gpg --decrypt sast-opengrep.sarif.gpg > sast-opengrep.sarif
```

Then open it in a SARIF viewer, see
[tutorials/01-setup-guide.md](../tutorials/01-setup-guide.md#step-5-read-the-results).

## Handling the password

Anyone holding `ARTIFACT_PASSWORD` can decrypt every report the pipeline has
ever published, and artifacts are retained for 30 days. Treat it as a shared
credential: keep the holders to the people who triage findings, and rotate it
when someone leaves the project. Rotation does not re-encrypt artifacts
already uploaded, so a rotation is only fully effective once the old ones
have expired.
