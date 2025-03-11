# Warehouse Label Maker

A Vue.js application for generating warehouse labels with QR codes. Perfect for organizing warehouse storage with a clear, standardized labeling system.

## Features

- Generate labels with rack, level, and bin numbers (e.g., 001 A 001)
- QR code generation with custom content support
- Configurable dimensions and weight capacity
- Optional directional arrows (up/down) with customizable position
- PDF export in A6 format (10x15 cm)
- Mobile-responsive interface

## Installation

```bash
# Install dependencies
npm install

# Start development server
npm run dev

# Build for production
npm run build
```

## Usage

1. Enter the rack number (format: 001)
2. Select the level (format: A)
3. Enter the bin number (format: 001)
4. Add optional dimensions and weight capacity
5. Add optional directional arrows
6. Optionally customize QR code content
7. Click "Generate PDF" to create a printable label

## Technologies Used

- Vue.js 3
- Tailwind CSS
- QR Code generation
- PDF export functionality

## License

MIT License
