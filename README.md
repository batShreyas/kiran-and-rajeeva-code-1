# kiran-and-rajeeva-code-1
kiran and rajeeva code 1


if (state->echoed && p && p->len >= 3 && memcmp(p->payload, "ACK", 3) == 0) {
    xil_printf("ACK received, closing.\n\r");
    pbuf_free(p);
    tcp_close(tpcb);
    return ERR_OK;
}


import socket
import os
import sys
import struct

PORT = 6001
CHUNK_SIZE = 1024
SEND_FILE = 'image_to_send.png'
RECEIVE_FILE = 'image_echoed_back.png'

def run_client(ip):
    if not os.path.exists(SEND_FILE):
        print(f"[-] '{SEND_FILE}' not found.")
        return

    # Step 1: Read image to send
    with open(SEND_FILE, 'rb') as f:
        image_data = f.read()
    image_size = len(image_data)
    print(f"[>] Sending '{SEND_FILE}' of size {image_size} bytes...")

    # Step 2: Connect to FPGA
    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
        s.connect((ip, PORT))
        print(f"[+] Connected to FPGA at {ip}:{PORT}")

        # Step 3: Send image size first
        s.sendall(struct.pack('!I', image_size))

        # Step 4: Send image
        s.sendall(image_data)
        s.shutdown(socket.SHUT_WR)
        print("[✓] Image sent. Waiting for echo...")

        # Step 5: Receive echoed image
        received_data = bytearray()
        expected_size = image_size

        while len(received_data) < expected_size:
            try:
                data = s.recv(CHUNK_SIZE)
                if not data:
                    print("❌ Connection closed by FPGA before full image received.")
                    break
                received_data.extend(data)
            except socket.error as e:
                print(f"❌ Socket recv error: {e}")
                break

        # Step 6: Check full reception
        if len(received_data) == expected_size:
            print(f"[✓] Echoed image fully received ({expected_size} bytes).")
        else:
            print(f"[!] Incomplete echo received: {len(received_data)} / {expected_size} bytes.")

        # Step 7: Save image
        with open(RECEIVE_FILE, 'wb') as f:
            f.write(received_data)
        print(f"[+] Echoed image saved as '{RECEIVE_FILE}'.")

        # Step 8: Send ACK to confirm reception
        s.sendall(b'ACK')  # FPGA can close after this
        print("[>] ACK sent to FPGA.")

if _name_ == '_main_':
    if len(sys.argv) != 2:
        print("Usage: python3 image_dual_peer_ack.py <FPGA_IP>")
        sys.exit(1)

    run_client(sys.argv[1])
