#!/usr/bin/env bash
#
# destroy_github_all.sh
# - Dry-run by default: daftar semua resource yang akan dihapus.
# - Untuk benar-benar menjalankan penghapusan, atur EXECUTE="YES_I_UNDERSTAND_DELETE_PERMANENTLY"
# - Requires: curl, jq
#
# Usage (dry-run):
#   GITHUB_TOKEN="ghp_xxx" ./destroy_github_all.sh
# To perform destructive actions:
#   GITHUB_TOKEN="ghp_xxx" EXECUTE="YES_I_UNDERSTAND_DELETE_PERMANENTLY" ./destroy_github_all.sh
#
set -euo pipefail

# Safety checks
if ! command -v curl >/dev/null 2>&1; then
  echo "Error: curl diperlukan. Install curl dan ulangi." >&2
  exit 2
fi
if ! command -v jq >/dev/null 2>&1; then
  echo "Error: jq diperlukan. Install jq dan ulangi." >&2
  exit 2
fi

: "${GITHUB_TOKEN:?GITHUB_TOKEN environment variable must be set (PAT with appropriate scopes)}"

API="https://api.github.com"
AUTH_HEADER="Authorization: token ${GITHUB_TOKEN}"
PER_PAGE=100

# Execution guard: must set EXECUTE env to the exact phrase to actually delete.
EXECUTE="${EXECUTE:-}"
EXECUTE_PHRASE="YES_I_UNDERSTAND_DELETE_PERMANENTLY"

is_execute_run() {
  [ "$EXECUTE" = "$EXECUTE_PHRASE" ]
}

echo "===== GitHub Mass-Delete Script (dry-run by default) ====="
echo "This script will operate on resources your token has permission to modify."
echo "If you REALLY want to proceed with permanent deletion, set:"
echo "  EXECUTE=\"$EXECUTE_PHRASE\""
echo

# Helper: paginated GET
paged_get() {
  local url="$1"
  local page=1
  while :; do
    resp=$(curl -sS -H "$AUTH_HEADER" "$url&per_page=$PER_PAGE&page=$page")
    if [ "$(echo "$resp" | jq 'length')" -eq 0 ]; then
      break
    fi
    echo "$resp"
    page=$((page+1))
  done
}

# 1) List user info (for safety)
user_login=$(curl -sS -H "$AUTH_HEADER" "$API/user" | jq -r .login)
if [ "$user_login" = "null" ] || [ -z "$user_login" ]; then
  echo "Gagal mengambil informasi user. Periksa token." >&2
  exit 3
fi
echo "Authenticated as: $user_login"
echo

# Confirm interactive prompt before any destructive run (extra guard)
if is_execute_run; then
  echo "EXECUTION MODE: WILL PERFORM DELETES (permanent)."
  echo "FINAL CHECK: Ketik nama akun Anda EXACTLY untuk melanjutkan: $user_login"
  read -r confirmname
  if [ "$confirmname" != "$user_login" ]; then
    echo "Nama tidak cocok. Dibatalkan."
    exit 1
  fi
else
  echo "DRY-RUN mode: tidak akan melakukan penghapusan. Untuk mengaktifkan penghapusan, set EXECUTE to $EXECUTE_PHRASE"
  echo
fi

# FUNCTIONS TO COLLECT AND (optionally) DELETE

# Delete repositories owned by the authenticated user
handle_repos() {
  echo "=== Checking repositories owned by $user_login ==="
  repos_json=$(curl -sS -H "$AUTH_HEADER" "$API/user/repos?type=owner&per_page=$PER_PAGE")
  repos=$(echo "$repos_json" | jq -r '.[].full_name')
  if [ -z "$repos" ]; then
    echo "(no owner repos found)"
    return
  fi

  echo "Repositories found (owner):"
  echo "$repos" | sed 's/^/ - /'
  echo

  if is_execute_run; then
    for r in $repos; do
      echo "Deleting repository: $r"
      resp_code=$(curl -s -o /dev/null -w "%{http_code}" -X DELETE -H "$AUTH_HEADER" "$API/repos/$r")
      if [ "$resp_code" -eq 204 ]; then
        echo "  OK: $r deleted"
      else
        echo "  WARNING: Failed to delete $r, HTTP $resp_code"
      fi
    done
  fi
}

# Delete gists
handle_gists() {
  echo "=== Checking Gists ==="
  gists_json=$(curl -sS -H "$AUTH_HEADER" "$API/gists?per_page=$PER_PAGE")
  gist_ids=$(echo "$gists_json" | jq -r '.[].id')
  if [ -z "$gist_ids" ]; then
    echo "(no gists found)"
    return
  fi
  echo "Gists found:"
  echo "$gist_ids" | sed 's/^/ - /'
  echo
  if is_execute_run; then
    for gid in $gist_ids; do
      echo "Deleting gist $gid"
      resp_code=$(curl -s -o /dev/null -w "%{http_code}" -X DELETE -H "$AUTH_HEADER" "$API/gists/$gid")
      if [ "$resp_code" -eq 204 ]; then
        echo "  OK: gist $gid deleted"
      else
        echo "  WARNING: Failed to delete gist $gid, HTTP $resp_code"
      fi
    done
  fi
}

# Delete public SSH keys for authenticated user
handle_user_keys() {
  echo "=== Checking SSH public keys for user ==="
  keys_json=$(curl -sS -H "$AUTH_HEADER" "$API/user/keys")
  key_ids=$(echo "$keys_json" | jq -r '.[].id')
  if [ -z "$key_ids" ]; then
    echo "(no user SSH keys found)"
    return
  fi
  echo "User SSH key IDs:"
  echo "$key_ids" | sed 's/^/ - /'
  echo
  if is_execute_run; then
    for kid in $key_ids; do
      echo "Deleting SSH key id=$kid"
      resp_code=$(curl -s -o /dev/null -w "%{http_code}" -X DELETE -H "$AUTH_HEADER" "$API/user/keys/$kid")
      if [ "$resp_code" -eq 204 ]; then
        echo "  OK: key $kid deleted"
      else
        echo "  WARNING: Failed to delete key $kid, HTTP $resp_code"
      fi
    done
  fi
}

# For each repo: delete deploy keys, webhooks, secrets, releases, artifacts
handle_repo_level() {
  echo "=== Scanning each owner repo for deploy-keys, webhooks, secrets and releases ==="
  repos_json=$(curl -sS -H "$AUTH_HEADER" "$API/user/repos?type=owner&per_page=$PER_PAGE")
  repos=$(echo "$repos_json" | jq -r '.[].full_name')
  if [ -z "$repos" ]; then
    echo "(no owner repos found)"
    return
  fi
  for r in $repos; do
    owner=$(echo "$r" | cut -d'/' -f1)
    repo=$(echo "$r" | cut -d'/' -f2)
    echo "-> Repo: $r"

    # Deploy keys
    dk_json=$(curl -sS -H "$AUTH_HEADER" "$API/repos/$owner/$repo/keys")
    dk_ids=$(echo "$dk_json" | jq -r '.[].id')
    if [ -n "$dk_ids" ]; then
      echo "   Deploy keys:"
      echo "$dk_ids" | sed 's/^/    - /'
      if is_execute_run; then
        for id in $dk_ids; do
          echo "    Deleting deploy key $id"
          resp_code=$(curl -s -o /dev/null -w "%{http_code}" -X DELETE -H "$AUTH_HEADER" "$API/repos/$owner/$repo/keys/$id")
          [ "$resp_code" -eq 204 ] && echo "      OK" || echo "      WARN HTTP $resp_code"
        done
      fi
    fi

    # Webhooks (hooks)
    hooks_json=$(curl -sS -H "$AUTH_HEADER" "$API/repos/$owner/$repo/hooks")
    hook_ids=$(echo "$hooks_json" | jq -r '.[].id')
    if [ -n "$hook_ids" ]; then
      echo "   Webhooks:"
      echo "$hook_ids" | sed 's/^/    - /'
      if is_execute_run; then
        for hid in $hook_ids; do
          echo "    Deleting hook $hid"
          resp_code=$(curl -s -o /dev
The names GitHub, GitHub Desktop, GitHub for Mac, GitHub for Windows, the Octocat, and related GitHub logos and/or stylized names are trademarks of GitHub. You agree not to display or use these trademarks in any manner without GitHub's prior, written permission, except as allowed by GitHub's Logos and Usage Policy: https://github.com/logos.

## Privacy

The Software may collect personal information. You may control what information the Software collects in the settings panel. If the Software does collect personal information on GitHub's behalf, GitHub will process that information in accordance with the [GitHub Privacy Statement](/site-policy/privacy-policies/github-privacy-statement).

## Additional Services

**Auto-Update Services**

The Software may include an auto-update service ("Service"). If you choose to use the Service or you download Software that automatically enables the Service, GitHub will automatically update the Software when a new version is available.

**Disclaimers and Limitations of Liability**

THE SERVICE IS PROVIDED ON AN "AS IS" BASIS, AND NO WARRANTY, EITHER EXPRESS OR IMPLIED, IS GIVEN. YOUR USE OF THE SERVICE IS AT YOUR SOLE RISK. GitHub does not warrant that (i) the Service will meet your specific requirements; (ii) the Service is fully compatible with any particular platform; (iii) your use of the Service will be uninterrupted, timely, secure, or error-free; (iv) the results that may be obtained from the use of the Service will be accurate or reliable; (v) the quality of any products, services, information, or other material purchased or obtained by you through the Service will meet your expectations; or (vi) any errors in the Service will be corrected.

YOU EXPRESSLY UNDERSTAND AND AGREE THAT GITHUB SHALL NOT BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, CONSEQUENTIAL OR EXEMPLARY DAMAGES, INCLUDING BUT NOT LIMITED TO, DAMAGES FOR LOSS OF PROFITS, GOODWILL, USE, DATA OR OTHER INTANGIBLE LOSSES (EVEN IF GITHUB HAS BEEN ADVISED OF THE POSSIBILITY OF SUCH DAMAGES) RELATED TO THE SERVICE, including, for example: (i) the use or the inability to use the Service; (ii) the cost of procurement of substitute goods and services resulting from any goods, data, information or services purchased or obtained or messages received or transactions entered into through or from the Service; (iii) unauthorized access to or alteration of your transmissions or data; (iv) statements or conduct of any third-party on the Service; (v) or any other matter relating to the Service.

GitHub reserves the right at any time and from time to time to modify or discontinue, temporarily or permanently, the Service (or any part thereof) with or without notice. GitHub shall not be liable to you or to any third-party for any price change, suspension or discontinuance of the Service.

## Miscellanea

1. No Waiver. The failure of GitHub to exercise or enforce any right or provision of these Application Terms shall not constitute a waiver of such right or provision.

1. Entire Agreement. These Application Terms, together with any applicable Privacy Notices, constitutes the entire agreement between you and GitHub and governs your use of the Software, superseding any prior agreements between you and GitHub (including, but not limited to, any prior versions of the Application Terms).

1. Governing Law. You agree that these Application Terms and your use of the Software are governed under California law and any dispute related to the Software must be brought in a tribunal of competent jurisdiction located in or near San Francisco, California.

1. Third-Party Packages. The Software supports third-party "Packages" which may modify, add, remove, or alter the functionality of the Software. These Packages are not covered by these Application Terms and may include their own license which governs your use of that particular package.

1. No Modifications; Complete Agreement. These Application Terms may only be modified by a written amendment signed by an authorized representative of GitHub, or by the posting by GitHub of a revised version. These Application Terms, together with any applicable Open Source Licenses and Notices and GitHub's Privacy Statement, represent the complete and exclusive statement of the agreement between you and us. These Application Terms supersede any proposal or prior agreement oral or written, and any other communications between you and GitHub relating to the subject matter of these terms.

1. License to GitHub Policies. These Application Terms are licensed under this [Creative Commons Zero license](https://creativecommons.org/publicdomain/zero/1.0/). For details, see our [site-policy repository](https://github.com/github/site-policy#license).

1. Contact Us. Questions about the Terms of Service? Contact us through the [GitHub Support portal](https://support.github.com/).
