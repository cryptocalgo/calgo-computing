#!/usr/bin/env python3
# MEC (Multi-access Edge Computing) Architecture Simulation

import random
import time
import uuid
from enum import Enum
from typing import Dict, List, Optional, Tuple


class TaskType(Enum):
    """Different types of tasks that can be processed in the system"""
    VIDEO_ANALYTICS = 1
    AR_VR = 2
    IOT_DATA = 3
    AUTONOMOUS_VEHICLE = 4


class Task:
    """Represents a computational task in the system"""
    def __init__(self, task_id: str, task_type: TaskType, data_size: float, 
                 processing_requirements: float, latency_requirement: float):
        self.task_id = task_id
        self.task_type = task_type
        self.data_size = data_size  # in MB
        self.processing_requirements = processing_requirements  # in GHz
        self.latency_requirement = latency_requirement  # in ms
        self.creation_time = time.time()
        self.completion_time: Optional[float] = None
        self.assigned_to: Optional[str] = None
    
    def calculate_latency(self) -> float:
        """Calculate the total latency for this task"""
        if self.completion_time is None:
            return float('inf')
        return (self.completion_time - self.creation_time) * 1000  # Convert to ms
    
    def __str__(self) -> str:
        return (f"Task {self.task_id} ({self.task_type.name}): "
                f"{self.data_size:.2f}MB, {self.processing_requirements:.2f}GHz, "
                f"Latency Req: {self.latency_requirement:.2f}ms")


class Device:
    """Represents an end-user device that generates tasks"""
    def __init__(self, device_id: str, location: Tuple[float, float]):
        self.device_id = device_id
        self.location = location  # (x, y) coordinates
        self.tasks_generated: List[Task] = []
    
    def generate_task(self) -> Task:
        """Generate a new task with random properties"""
        task_type = random.choice(list(TaskType))
        
        # Task properties based on type
        if task_type == TaskType.VIDEO_ANALYTICS:
            data_size = random.uniform(5.0, 20.0)  # Larger data size
            processing_req = random.uniform(1.0, 3.0)
            latency_req = random.uniform(50.0, 200.0)
        elif task_type == TaskType.AR_VR:
            data_size = random.uniform(1.0, 5.0)
            processing_req = random.uniform(2.0, 4.0)
            latency_req = random.uniform(10.0, 50.0)  # Strict latency requirement
        elif task_type == TaskType.IOT_DATA:
            data_size = random.uniform(0.1, 1.0)  # Small data size
            processing_req = random.uniform(0.2, 1.0)
            latency_req = random.uniform(100.0, 500.0)
        else:  # AUTONOMOUS_VEHICLE
            data_size = random.uniform(1.0, 10.0)
            processing_req = random.uniform(3.0, 5.0)
            latency_req = random.uniform(5.0, 20.0)  # Very strict latency requirement
        
        task_id = f"task_{uuid.uuid4().hex[:8]}"
        task = Task(task_id, task_type, data_size, processing_req, latency_req)
        self.tasks_generated.append(task)
        return task
    
    def __str__(self) -> str:
        return f"Device {self.device_id} at location {self.location}"


class ComputeNode:
    """Base class for any computing node (MEC Host or Cloud)"""
    def __init__(self, node_id: str, processing_capacity: float, 
                 location: Optional[Tuple[float, float]] = None):
        self.node_id = node_id
        self.processing_capacity = processing_capacity  # in GHz
        self.location = location
        self.current_load = 0.0
        self.tasks_queue: List[Task] = []
        self.completed_tasks: List[Task] = []
    
    def can_accept_task(self, task: Task) -> bool:
        """Check if the node has capacity to accept a new task"""
        return self.current_load + task.processing_requirements <= self.processing_capacity
    
    def accept_task(self, task: Task) -> bool:
        """Accept a task for processing if capacity allows"""
        if self.can_accept_task(task):
            self.tasks_queue.append(task)
            self.current_load += task.processing_requirements
            task.assigned_to = self.node_id
            return True
        return False
    
    def process_tasks(self, network_latency: float = 0.0) -> List[Task]:
        """Process tasks in the queue and return completed tasks"""
        completed = []
        
        for task in self.tasks_queue[:]:
            # Simulate processing time (simplified)
            processing_time = task.processing_requirements / self.processing_capacity
            
            # Calculate total latency including network latency
            total_latency = (processing_time * 1000) + network_latency  # Convert to ms
            
            # If task would complete in this cycle
            task.completion_time = time.time()
            self.current_load -= task.processing_requirements
            self.tasks_queue.remove(task)
            self.completed_tasks.append(task)
            completed.append(task)
        
        return completed
    
    def get_utilization(self) -> float:
        """Return current utilization as a percentage"""
        if self.processing_capacity == 0:
            return 0.0
        return (self.current_load / self.processing_capacity) * 100
    
    def __str__(self) -> str:
        return (f"{self.__class__.__name__} {self.node_id}: "
                f"Capacity: {self.processing_capacity:.2f}GHz, "
                f"Load: {self.current_load:.2f}GHz ({self.get_utilization():.2f}%)")


class MECHost(ComputeNode):
    """Represents a MEC Host that processes tasks at the edge"""
    def __init__(self, node_id: str, processing_capacity: float, 
                 location: Tuple[float, float], coverage_radius: float):
        super().__init__(node_id, processing_capacity, location)
        self.coverage_radius = coverage_radius  # in distance units
    
    def is_device_in_coverage(self, device: Device) -> bool:
        """Check if a device is within this MEC Host's coverage area"""
        if self.location is None:
            return False
        
        x1, y1 = self.location
        x2, y2 = device.location
        distance = ((x2 - x1) ** 2 + (y2 - y1) ** 2) ** 0.5
        return distance <= self.coverage_radius


class CloudDataCenter(ComputeNode):
    """Represents a cloud data center with high capacity but higher latency"""
    def __init__(self, node_id: str, processing_capacity: float):
        # Cloud doesn't have a specific location
        super().__init__(node_id, processing_capacity)


class MECPlatform:
    """The MEC Platform that manages MEC hosts and orchestrates task distribution"""
    def __init__(self):
        self.mec_hosts: Dict[str, MECHost] = {}
        self.cloud = CloudDataCenter("cloud_dc", 1000.0)  # Very high capacity
        self.devices: Dict[str, Device] = {}
        self.network_latency_to_cloud = 100.0  # ms
    
    def add_mec_host(self, mec_host: MECHost) -> None:
        """Add a MEC Host to the platform"""
        self.mec_hosts[mec_host.node_id] = mec_host
    
    def add_device(self, device: Device) -> None:
        """Register a device with the platform"""
        self.devices[device.device_id] = device
    
    def calculate_network_latency(self, source_location: Tuple[float, float],
                                 target_location: Optional[Tuple[float, float]]) -> float:
        """Calculate network latency based on distance (simplified model)"""
        if target_location is None:  # For cloud, use fixed latency
            return self.network_latency_to_cloud
        
        x1, y1 = source_location
        x2, y2 = target_location
        distance = ((x2 - x1) ** 2 + (y2 - y1) ** 2) ** 0.5
        
        # Simple model: 1 distance unit = 1ms latency (minimum 1ms)
        return max(1.0, distance * 1.0)
    
    def find_best_mec_host(self, device: Device, task: Task) -> Optional[MECHost]:
        """Find the best MEC Host for a task based on location and capacity"""
        eligible_hosts = []
        
        for host in self.mec_hosts.values():
            if not host.is_device_in_coverage(device):
                continue
                
            if not host.can_accept_task(task):
                continue
                
            # Calculate network latency between device and host
            latency = self.calculate_network_latency(device.location, host.location)
            
            # Check if the host can meet latency requirements
            if latency <= task.latency_requirement:
                eligible_hosts.append((host, latency))
        
        if not eligible_hosts:
            return None
            
        # Sort by latency (lowest first)
        eligible_hosts.sort(key=lambda x: x[1])
        return eligible_hosts[0][0]
    
    def orchestrate_task(self, device: Device, task: Task) -> str:
        """Decide where to process a task and assign it to the node"""
        # First try to find a suitable MEC Host
        best_host = self.find_best_mec_host(device, task)
        
        if best_host is not None:
            if best_host.accept_task(task):
                return f"Task {task.task_id} assigned to MEC Host {best_host.node_id}"
        
        # If no suitable MEC Host, try cloud if it meets latency requirements
        cloud_latency = self.calculate_network_latency(device.location, None)
        if cloud_latency <= task.latency_requirement and self.cloud.accept_task(task):
            return f"Task {task.task_id} assigned to Cloud"
        
        return f"Task {task.task_id} could not be assigned (requirements too strict)"
    
    def run_simulation_cycle(self) -> List[str]:
        """Run one cycle of the simulation"""
        results = []
        
        # Process tasks at MEC Hosts
        for host_id, host in self.mec_hosts.items():
            completed_tasks = host.process_tasks()
            for task in completed_tasks:
                device = self.devices[task.task_id.split('_')[1]] if '_' in task.task_id else None
                latency = task.calculate_latency()
                results.append(f"Task {task.task_id} completed at MEC Host {host_id} with latency {latency:.2f}ms")
        
        # Process tasks at Cloud
        cloud_completed = self.cloud.process_tasks(self.network_latency_to_cloud)
        for task in cloud_completed:
            latency = task.calculate_latency()
            results.append(f"Task {task.task_id} completed at Cloud with latency {latency:.2f}ms")
        
        return results
    
    def generate_summary(self) -> Dict:
        """Generate a summary of the platform state"""
        mec_utilization = {host_id: host.get_utilization() for host_id, host in self.mec_hosts.items()}
        cloud_utilization = self.cloud.get_utilization()
        
        # Completed tasks statistics
        mec_completed = sum(len(host.completed_tasks) for host in self.mec_hosts.values())
        cloud_completed = len(self.cloud.completed_tasks)
        
        # Calculate average latency
        all_completed = []
        for host in self.mec_hosts.values():
            all_completed.extend(host.completed_tasks)
        all_completed.extend(self.cloud.completed_tasks)
        
        if all_completed:
            avg_latency = sum(task.calculate_latency() for task in all_completed) / len(all_completed)
        else:
            avg_latency = 0
        
        return {
            "mec_utilization": mec_utilization,
            "cloud_utilization": cloud_utilization,
            "mec_completed_tasks": mec_completed,
            "cloud_completed_tasks": cloud_completed,
            "total_completed_tasks": mec_completed + cloud_completed,
            "average_latency": avg_latency
        }


def run_demo():
    """Run a demonstration of the MEC Architecture simulation"""
    # Create MEC Platform
    platform = MECPlatform()
    
    # Create MEC Hosts
    mec1 = MECHost("mec_host_1", 10.0, (0, 0), 5.0)
    mec2 = MECHost("mec_host_2", 8.0, (10, 0), 5.0)
    mec3 = MECHost("mec_host_3", 12.0, (5, 8), 6.0)
    
    platform.add_mec_host(mec1)
    platform.add_mec_host(mec2)
    platform.add_mec_host(mec3)
    
    # Create Devices
    devices = [
        Device(f"device_{i}", (random.uniform(0, 15), random.uniform(0, 10)))
        for i in range(1, 11)  # 10 devices
    ]
    
    for device in devices:
        platform.add_device(device)
        print(f"Created {device}")
    
    print("\n--- Starting Simulation ---\n")
    
    # Run for 5 cycles
    for cycle in range(1, 6):
        print(f"\n--- Cycle {cycle} ---")
        
        # Generate random tasks from devices
        for device in random.sample(devices, 3):  # 3 random devices generate tasks each cycle
            task = device.generate_task()
            print(f"{device.device_id} generated {task}")
            result = platform.orchestrate_task(device, task)
            print(result)
        
        # Process tasks
        print("\nProcessing tasks...")
        results = platform.run_simulation_cycle()
        for result in results:
            print(result)
        
        # Print summary
        if cycle == 5:
            print("\n--- Final Summary ---")
            summary = platform.generate_summary()
            print(f"MEC Host Utilization: {summary['mec_utilization']}")
            print(f"Cloud Utilization: {summary['cloud_utilization']:.2f}%")
            print(f"Tasks Completed at MEC: {summary['mec_completed_tasks']}")
            print(f"Tasks Completed at Cloud: {summary['cloud_completed_tasks']}")
            print(f"Total Completed Tasks: {summary['total_completed_tasks']}")
            print(f"Average Latency: {summary['average_latency']:.2f}ms")


if __name__ == "__main__":
    run_demo()