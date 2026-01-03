# Data Format

The application loads data from `/public/data.json`.

## JSON Format

The data file should be a JSON object with two arrays: `data` and `events`.

```json
{
  "data": [
    {
      "timestamp": "2024-01-15",
      "x": -50,
      "y": 10,
      "z": 0,
      "color": "#ff6b6b",
      "count": 145,
      "label": "Server Group A",
      "shape": "sphere",
      "size": 2
    }
  ],
  "events": [
    {
      "timestamp": "2024-02-15",
      "title": "New SLA Policy",
      "description": "24hr response time for critical findings"
    }
  ]
}
```

## Field Descriptions

### Data Points

| Field | Type | Description |
|-------|------|-------------|
| `timestamp` | string | ISO 8601 date string (YYYY-MM-DD) |
| `x` | number | X coordinate in 3D space |
| `y` | number | Y coordinate in 3D space |
| `z` | number | Z coordinate in 3D space |
| `color` | string | Hex color code (e.g., "#ff6b6b") |
| `count` | number | Quantitative value for timeline bar height (can be negative) |
| `label` | string | Display name/category for the data point |
| `shape` | string | (Optional) Shape of the 3D object. Default: "sphere" |
| `size` | number | (Optional) Size of the 3D object. Default: 2 |

#### Available Shapes

- `sphere` - Spherical object (default)
- `box` or `cube` - Cubic object
- `cone` - Conical object
- `cylinder` - Cylindrical object
- `torus` - Donut/ring shaped object
- `octahedron` - 8-sided polyhedron
- `tetrahedron` - 4-sided polyhedron (pyramid)
- `dodecahedron` - 12-sided polyhedron
- `icosahedron` - 20-sided polyhedron

### Events

| Field | Type | Description |
|-------|------|-------------|
| `timestamp` | string | ISO 8601 date string (YYYY-MM-DD) |
| `title` | string | Event title (short name) |
| `description` | string | Detailed description of the event |

## Features

### Data Points
- Displayed as 3D objects in the visualization (shape and size customizable)
- Shown as bars on the timeline
- Listed in the data table with full details
- Clickable in both table and timeline to animate camera focus

### Events
- Displayed as red circular markers on the timeline
- Show tooltip on hover with title and description
- Listed in a separate events table below the data table
- Do not appear in the 3D space

## How to Update Data

1. Edit `/public/data.json` with your data
2. Ensure all timestamps are in `YYYY-MM-DD` format
3. The `events` array is optional - you can omit it if you have no events
4. The application will automatically load the data on refresh
5. If the file is not found or has errors, the application falls back to sample data

## Example

See `/public/data.json` for a complete example with 5 data points and 2 events spanning January to May 2024.
