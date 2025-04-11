import hashlib
import time
import random
import struct

def calculate_hash(version, prev_block, merkle_root, timestamp, bits, nonce):
    header = (
        struct.pack("<L", version) +
        bytes.fromhex(prev_block)[::-1] +
        bytes.fromhex(merkle_root)[::-1] +
        struct.pack("<L", timestamp) +
        struct.pack("<L", bits) +
        struct.pack("<L", nonce)
    )
    return hashlib.sha256(hashlib.sha256(header).digest()).digest()[::-1].hex()

def mine_bitcoin_block(difficulty_bits):
    # In the actual Bitcoin network, these values are fetched from the network
    version = 2
    prev_block = "000000000000000000046b9e5a11c078c37fa8e6e89cca9fcfeec5c27223a839"
    merkle_root = "4a5e1e4baab89f3a32518a88c31bc87f618f76673e2cc77ab2127b7afdeda33b"
    timestamp = int(time.time())
    bits = 0x1d00ffff  # Difficulty target (actually determined by the network)
    
    # Set target difficulty (set to a low value to find a hash quickly in this example)
    target = 2 ** (256 - difficulty_bits)
    
    print(f"Starting Bitcoin mining...")
    print(f"Target: Find a hash with {difficulty_bits} leading zero bits")
    
    max_nonce = 2 ** 32  # Maximum nonce value
    nonce = random.randint(0, max_nonce)
    start_time = time.time()
    
    for _ in range(max_nonce):
        hash_result = calculate_hash(version, prev_block, merkle_root, timestamp, bits, nonce)
        
        # Convert hash to integer to compare with target
        hash_int = int(hash_result, 16)
        
        if hash_int < target:
            elapsed_time = time.time() - start_time
            hash_rate = nonce / elapsed_time if elapsed_time > 0 else 0
            
            print(f"\nSuccess! Found valid hash on attempt {nonce}")
            print(f"Elapsed time: {elapsed_time:.2f} seconds")
            print(f"Hash rate: {hash_rate:.2f} hashes/second") 
            print(f"Block hash: {hash_result}")
            return hash_result
        
        nonce += 1
        if nonce >= max_nonce:
            print("Maximum attempt count reached. Mining failed.")
            break
        
        # Display progress
        if nonce % 100000 == 0:
            print(f"Trying: {nonce} hashes calculated...", end="\r")
    
    return None

if __name__ == "__main__":
    # Set difficulty to a low value (actual Bitcoin is much higher)
    # Higher value = more difficult, lower value = easier
    difficulty = 10  # Set low for educational purposes
    mine_bitcoin_block(difficulty)