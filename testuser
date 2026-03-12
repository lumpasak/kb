#!/bin/bash
# create_user.sh - Create a new Linux user with common options

set -euo pipefail

# Check root privileges
if [[ $EUID -ne 0 ]]; then
    echo "Error: This script must be run as root (use sudo)." >&2
    exit 1
fi

# Usage
usage() {
    echo "Usage: $0 -u USERNAME [-c COMMENT] [-s SHELL] [-g GROUPS] [-e] [-h]"
    echo ""
    echo "Options:"
    echo "  -u USERNAME   Login name (required)"
    echo "  -c COMMENT    Full name / description"
    echo "  -s SHELL      Login shell (default: /bin/bash)"
    echo "  -g GROUPS     Comma-separated supplementary groups"
    echo "  -e            Force password change on first login"
    echo "  -h            Show this help"
    exit 1
}

# Defaults
SHELL_PATH="/bin/bash"
COMMENT=""
GROUPS=""
EXPIRE_PW=false

while getopts "u:c:s:g:eh" opt; do
    case $opt in
        u) USERNAME="$OPTARG" ;;
        c) COMMENT="$OPTARG" ;;
        s) SHELL_PATH="$OPTARG" ;;
        g) GROUPS="$OPTARG" ;;
        e) EXPIRE_PW=true ;;
        h) usage ;;
        *) usage ;;
    esac
done

# Validate username
if [[ -z "${USERNAME:-}" ]]; then
    echo "Error: Username is required (-u)." >&2
    usage
fi

if id "$USERNAME" &>/dev/null; then
    echo "Error: User '$USERNAME' already exists." >&2
    exit 1
fi

# Build useradd command
CMD=(useradd -m -s "$SHELL_PATH")

[[ -n "$COMMENT" ]] && CMD+=(-c "$COMMENT")
[[ -n "$GROUPS" ]]  && CMD+=(-G "$GROUPS")

CMD+=("$USERNAME")

# Create the user
echo "Creating user '$USERNAME'..."
"${CMD[@]}"

# Set password interactively
echo "Set password for '$USERNAME':"
passwd "$USERNAME"

# Expire password if requested
if $EXPIRE_PW; then
    chage -d 0 "$USERNAME"
    echo "Password will expire on first login."
fi

# Summary
echo ""
echo "=== User created successfully ==="
echo "Username : $USERNAME"
echo "Home     : /home/$USERNAME"
echo "Shell    : $SHELL_PATH"
[[ -n "$GROUPS" ]] && echo "Groups   : $GROUPS"
echo "================================="
