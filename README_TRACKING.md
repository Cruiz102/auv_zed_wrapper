# Static Object Tracking Configuration

## Overview

The ZED custom wrapper now uses **distance-based object tracking** optimized for **static objects**. This is ideal for mapping fixed objects in an environment like furniture, appliances, or stationary equipment.

**Important**: Only objects listed in `expected_object_counts` will be added to the map and shown in RViz markers. All other detected classes are filtered out.

## How It Works

### 1. Distance-Based Matching
- When a new object is detected, the system first checks if its class is in `expected_object_counts`
- **If not listed**: The detection is ignored (filtered out)
- **If listed**: The system searches for existing objects of the **same class**
- If an existing object is found within the `matching_distance_threshold`, it's considered the **same object** and its position is updated
- If no match is found within the threshold, it's considered a **new object**

### 2. Expected Object Counts (Required)
- You **must** specify which classes to track and how many of each you expect
- **Only these classes will appear in the map and RViz markers**
- This prevents false detections or duplicates from being added to the map
- If the expected count is reached, additional detections of that class are ignored

## Configuration Parameters

### `matching_distance_threshold` (default: 1.0)
- **Type**: float (meters)
- **Description**: Maximum distance for considering two detections as the same object
- **Recommendation**: 
  - Use **0.5-1.0m** for small objects (chairs, bottles, books)
  - Use **1.0-2.0m** for larger objects (tables, couches, beds)
  - Use smaller values for more precise tracking

### `expected_object_counts` (Required for filtering)
- **Type**: array of integers `[class_id, count, class_id, count, ...]`
- **Description**: Expected number of objects for each class **and which classes to track**
- **Format**: Pairs of (YOLO class ID, expected count)
- **Important**: Only classes listed here will be tracked and published to the map/markers
- **Empty list behavior**: If empty `[]`, NO objects will be added to the map (all detections filtered)

#### YOLO COCO Class IDs (common examples):
```
0  = person
56 = chair
57 = couch
58 = potted plant
59 = bed
60 = dining table
62 = tv
63 = laptop
67 = cell phone
73 = book
74 = clock
```

## Example Configurations

### Example 1: Living Room
```python
parameters=[
    {'matching_distance_threshold': 1.0},
    {'expected_object_counts': [
        57, 1,   # 1 couch
        56, 4,   # 4 chairs
        62, 1,   # 1 TV
        60, 1    # 1 dining table
    ]}
]
```

### Example 2: Office
```python
parameters=[
    {'matching_distance_threshold': 0.8},
    {'expected_object_counts': [
        56, 6,   # 6 chairs (around meeting table)
        60, 1,   # 1 table
        63, 3,   # 3 laptops
        73, 10   # 10 books
    ]}
]
```

### Example 3: Kitchen
```python
parameters=[
    {'matching_distance_threshold': 1.2},
    {'expected_object_counts': [
        56, 4,   # 4 chairs
        67, 2,   # 2 cell phones
        60, 1,   # 1 dining table
        66, 1    # 1 refrigerator
    ]}
]
```

## Updating Launch File

Edit `launch/zed_custom.launch.py`:

```python
Node(
    package='zed_custom_wrapper',
    executable='zed_custom_node',
    name='zed_custom_node',
    output='screen',
    parameters=[
        # ... other parameters ...
        {'matching_distance_threshold': 1.0},
        {'expected_object_counts': [56, 4, 62, 1, 60, 1]}  # 4 chairs, 1 TV, 1 table
    ]
)
```

## Behavior

### Class Filtering:
- **Only classes in `expected_object_counts` are tracked**
- All other detected objects are ignored (logged at DEBUG level)
- This keeps your map clean with only relevant objects

### When Objects Are Detected:
1. **Check if class is in expected_object_counts** → If not, ignore
2. **First detection** of an expected class → Added to map with new index
3. **Subsequent detection** within threshold → Updates existing object position
4. **Detection beyond threshold** → Added as new object (if count not exceeded)
5. **Exceeds expected count** → Warning logged, detection ignored

### Log Messages:
- `Ignoring detected object of class X (not in expected_object_counts)` → Class filtered out
- `New static object: Index=X, Class=Y, Position=[x, y, z]` → New object added
- `Updated existing object at index X, Class=Y` → Object position updated
- `Detected object of class X but already have Y/Z expected` → Count limit reached

## Tips

1. **Always specify expected_object_counts**: If empty, no objects will be tracked!
2. **Start with known objects**: List only the objects you want to track in your scene
3. **Tune threshold**: Adjust `matching_distance_threshold` based on your camera movement and object sizes
4. **Monitor logs**: Check ROS logs to see which objects are being tracked or filtered
5. **Visualize in RViz**: Use the `/zed/map_markers` topic to see only your tracked objects with their IDs

## Troubleshooting

**Problem**: Duplicate objects in the map
- **Solution**: Decrease `matching_distance_threshold`

**Problem**: Objects not being added
- **Solution**: Increase `matching_distance_threshold` or remove expected count limits

**Problem**: Too many false detections
- **Solution**: Set appropriate `expected_object_counts` for your scene
