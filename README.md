# JalwaDB
An LSM based Key Value Store
not import os
import struct
from typing import Dict, Tuple, List

class Entry:
    def __init__(self, key: bytes, value: bytes, timestamp: int):
        self.key = key
        self.value = value
        self.timestamp = timestamp

class MemTable:
    def __init__(self):
        self.data: Dict[bytes, Entry] = {}
    
    def insert(self, key: bytes, value: bytes):
        entry = Entry(key, value, time.time())
        self.data[key] = entry
    
    def lookup(self, key: bytes) -> Entry | None:
        return self.data.get(key)

class SSTable:
    MAGIC_NUMBER = b'JALWADB'
    
    def __init__(self, level: int, index: int, path: str):
        self.level = level
        self.index = index
        self.path = f"{path}/L{level}-{index}.sst"
    
    @staticmethod
    def serialize(entries: List[Entry]) -> bytes:
        data = SSTable.MAGIC_NUMBER
        data += struct.pack('<I', len(entries))
        
        serialized_entries = []
        last_key = None
        
        for entry in entries:
            if last_key is None or not entry.key.startswith(last_key):
                # Store whole key
                serialized_entries.append((entry.key, len(entry.key)))
                last_key = entry.key
            else:
                # Store only suffix
                prefix_len = common_prefix_length(entry.key, last_key)
                serialized_entries.append((
                    entry.key[prefix_len:], 
                    len(entry.key) - prefix_len,
                    prefix_len
                ))
            
            serialized_entries.append((struct.pack('<Q', entry.timestamp), 8))
            serialized_entries.append((struct.pack('B', len(entry.value)), 1))
            serialized_entries.append((entry.value, len(entry.value)))
            
        return b''.join(serialized_entries)
    
    @staticmethod
    def deserialize(data: bytes) -> List[Entry]:
        magic = data[:len(SSTable.MAGIC_NUMBER)]
        assert magic == SSTable.MAGIC_NUMBER
        
        entry_count = struct.unpack('<I', data[len(magic):len(magic)+4])[0]
        offset = len(magic) + 4
        
        entries = []
        last_key = None
        
        for _ in range(entry_count):
            key_len_raw = struct.unpack('<H', data[offset:offset+2])[0]
            offset += 2
            
            if key_len_raw & 0x8000:
                # Only suffix stored
                prefix_len = key_len_raw & 0x7FFF
                key_suffix_bytes = data[offset:offset+key_len_raw-prefix_len]
                key = last_key[:prefix_len] + key_suffix_bytes
                offset += key_len_raw - prefix_len
            else:
                # Full key stored
                key_bytes = data[offset:offset+key_len_raw]
                key = key_bytes
                offset += key_len_raw
                
            timestamp_bytes = data[offset:offset+8]
            timestamp = struct.unpack('<Q', timestamp_bytes)[0]
            offset += 8
            
            value_len = struct.unpack('B', data[offset:offset+1])[0]
            offset += 1
            
            value_bytes = data[offset:offset+value_len]
            offset += value_len
            
            entries.append(Entry(key, value_bytes, timestamp))
            last_key = key
            
        return entries

def common_prefix_length(s1: bytes, s2: bytes) -> int:
    min_len = min(len(s1), len(s2))
    for i in range(min_len):
        if s1[i] != s2[i]:
            return i
    return min_len
