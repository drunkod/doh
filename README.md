# Nix-based DNS-over-HTTPS (DoH) Server with Blocky and Nginx

This project sets up a DNS-over-HTTPS (DoH) server using Blocky as the DNS resolver and Nginx as the DoH proxy. The environment is defined using Nix and is configured to run within a cloud IDE (like Google Cloud Workstations) or any environment where Nix is available.

## Features

*   **Blocky:** DNS resolver with ad-blocking capabilities (minimal config by default).
*   **Nginx:** Acts as a reverse proxy, terminating TLS (if applicable at the edge) and forwarding DoH requests to Blocky.
*   **Nix Environment:** Reproducible development and runtime environment.
*   **Automated Startup & Test:** The `dev.nix` includes an `onStart` hook that:
    *   Automatically starts Blocky.
    *   Automatically starts Nginx.
    *   Runs an internal `curl` test to verify the DoH setup.
*   **Publicly Accessible (via Cloud IDE Port Forwarding):** Configured to be testable from the public internet when port forwarding is set to "public" in the cloud IDE.

## Prerequisites

*   A Nix-enabled environment (e.g., a Linux machine with Nix installed, or a cloud IDE with Nix support).
*   `curl`, `xxd`, `base64`, `tr` for testing.
*   If testing externally from a cloud IDE, the relevant port (default 8080) must be forwarded and made publicly accessible from the IDE's settings.

## Setup and Usage (within the Nix Environment / Cloud IDE)

1.  **Build the Environment:**
    If you have this project cloned and your cloud IDE uses `dev.nix` (or `.idx/dev.nix`), the environment should build automatically when the workspace loads. If you need to trigger a rebuild, consult your IDE's documentation (e.g., "IDX: Rebuild Nix Environment").

2.  **Automated Startup (`onStart` Hook):**
    When the workspace starts or restarts, the `onStart` hook in `dev.nix` will:
    *   Kill any pre-existing Blocky and Nginx instances (started by previous `onStart` runs).
    *   Start Blocky in the background.
        *   Configuration: Defined in `dev.nix` (`minimalBlockyConfig`).
        *   Logs: `/tmp/blocky.log`
        *   Serves DoH internally on `http://localhost:4000/dns-query`.
    *   Start Nginx in the background.
        *   Configuration: Defined in `dev.nix` (`fixedPortNginxConfig`).
        *   Logs: `/tmp/nginx-onstart.log`
        *   PID File: `/tmp/nginx-onstart.pid`
        *   Listens on `http://localhost:8080` and proxies `/dns-query` to Blocky.
    *   Run an automated `curl` test to verify the internal DoH functionality. Check the terminal output for success messages.

3.  **Log Files:**
    *   Blocky logs: `cat /tmp/blocky.log`
    *   Nginx (started by `onStart`) logs: `cat /tmp/nginx-onstart.log`

## Testing the DoH Server

You can test the DoH server both from within the Nix environment and externally (if port forwarding is public).

### 1. Internal Testing (within the Nix Environment / Cloud IDE Terminal)

The `onStart` hook already performs an automated test. You can also run it manually:

*   **Prepare the DNS Query Message:**
    This example creates a query for an A record for `example.com`.
    ```bash
    DNS_QUERY_HEX="000101000001000000000000076578616d706c6503636f6d0000010001"
    DNS_QUERY_MSG=$(echo -n $DNS_QUERY_HEX | xxd -r -p | base64 | tr -- '+/' '-_' | tr -d '=')
    echo "Generated DNS_QUERY_MSG: $DNS_QUERY_MSG" 
    # Expected output: AAEBAAABAAAAAAAAB2V4YW1wbGUDY29tAAABAAE
    ```

*   **Execute the DoH Query using `curl`:**
    ```bash
    # Request 'application/dns-message' and view hex output
    curl -v "http://localhost:8080/dns-query?dns=$DNS_QUERY_MSG" \
      -H 'accept: application/dns-message' | od -t x1 -A n

    # Request 'application/dns-json' (if you want to see if Blocky/Nginx return JSON)
    # and pretty-print with jq
    curl -s "http://localhost:8080/dns-query?dns=$DNS_QUERY_MSG" \
      -H 'accept: application/dns-json' | jq .
    ```
    A successful query will return an HTTP 200 status code and the DNS response.

### 2. External Testing (from your Local Notebook via Public URL)

This assumes:
*   Your cloud IDE is forwarding the internal port 8080 to a public URL.
*   You have configured this forwarded port to be publicly accessible (unauthenticated).
*   Replace `YOUR_PUBLIC_DOH_URL` with the actual URL provided by your cloud IDE (e.g., `https://8080-your-instance.cloudworkstations.dev`).

*   **Prepare the DNS Query Message (as above on your local machine).**

*   **Execute the DoH Query using `curl`:**
    ```bash
    # Replace YOUR_PUBLIC_DOH_URL with the actual public URL
    # Request 'application/dns-message'
    curl -v "YOUR_PUBLIC_DOH_URL/dns-query?dns=$DNS_QUERY_MSG" \
      -H 'accept: application/dns-message' | od -t x1 -A n

    # Request 'application/dns-json'
    curl -s "YOUR_PUBLIC_DOH_URL/dns-query?dns=$DNS_QUERY_MSG" \
      -H 'accept: application/dns-json' | jq .
    ```

*   **Testing with a Browser:**
    1.  Accessing `YOUR_PUBLIC_DOH_URL/` in a browser should result in a **403 Forbidden** error, as configured in Nginx. This is expected.
    2.  To inspect a DoH query via the browser:
        *   Open Developer Tools (F12) and go to the Network tab.
        *   Construct the full DoH query URL in the address bar:
            `YOUR_PUBLIC_DOH_URL/dns-query?dns=AAEBAAABAAAAAAAAB2V4YW1wbGUDY29tAAABAAE`
        *   Observe the request in the Network tab. It should return HTTP 200 with the DNS response. The browser might try to download the `application/dns-message` payload.

*   **Using a DoH Client:**
    You can configure DoH-aware applications, browsers (like Firefox or Chrome with secure DNS settings), or operating systems (if supported) to use your DoH server URL:
    `YOUR_PUBLIC_DOH_URL/dns-query`

## Configuration Files

*   **`dev.nix`:** Defines the Nix environment, packages, Nginx configuration, Blocky configuration, and startup scripts.
*   **Blocky Configuration (`minimalBlockyConfig` in `dev.nix`):**
    *   Listens for DoH on `http://0.0.0.0:4000`.
    *   Plain DNS is disabled (`port: 0`).
    *   Uses public upstream DNS servers (1.1.1.1, 8.8.8.8).
*   **Nginx Configuration (`fixedPortNginxConfig` in `dev.nix`):**
    *   Listens on `http://localhost:8080`.
    *   Proxies requests from `/dns-query` to `http://localhost:4000/dns-query` (Blocky).
    *   Denies all other requests (e.g., to `/`).

## Troubleshooting

*   **Service not starting:**
    *   Check `onStart` script output in the IDE terminal.
    *   Inspect Blocky logs: `cat /tmp/blocky.log`
    *   Inspect Nginx logs: `cat /tmp/nginx-onstart.log`
*   **DoH queries failing:**
    *   Ensure both Blocky and Nginx are running.
    *   Use `curl -v` for verbose output to see detailed request/response headers and errors.
    *   Verify the `DNS_QUERY_MSG` is correctly generated.
    *   If testing externally, ensure your public URL is correct and the port forwarding is configured for public access. If you get 302 redirects, the port might still be behind an authentication layer from the cloud IDE.

## Further Development

*   **Customize Blocky:** Modify `minimalBlockyConfig` in `dev.nix` to add blocklists, conditional forwarding, client groups, etc.
*   **Secure Nginx:** For a production setup, you would typically have Nginx handle TLS termination (HTTPS) directly. The cloud IDE's forwarding often provides HTTPS at the edge.
*   **Add more robust health checks:** The `sleep` in `onStart` is basic. More advanced checks could poll the service endpoints.
*   **Persistent Logging:** Redirect logs to persistent storage if needed.

