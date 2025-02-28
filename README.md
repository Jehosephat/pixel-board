# GalaChain Burn dApp

A dApp for burning and transferring GALA tokens, featuring an interactive community pixel board.

## Features

- Connect wallet and burn GALA tokens
- Transfer GALA tokens to other users
- Interactive Community Pixel Board
  - Draw pixels by burning GALA tokens
  - Real-time updates across all users
  - Each pixel costs 1 GALA to place
  - Shared canvas persisted in Firebase

## Setup

1. Clone the repository
2. Install dependencies: `npm install`
3. Create a Firebase project:
   - Go to [Firebase Console](https://console.firebase.google.com/)
   - Create a new project
   - Add a web app to get your config
   - Create a Realtime Database
4. Create a `.env` file with your configuration:
   ```env
   VITE_BURN_GATEWAY_API=https://gateway-mainnet.galachain.com/api/asset/token-contract
   VITE_BURN_GATEWAY_PUBLIC_KEY_API=https://api-galaswap.gala.com/galachain/api/asset/public-key-contract/GetPublicKey
   VITE_GALASWAP_API=https://api-galaswap.gala.com/galachain
   VITE_PROJECT_ID=your_project_id

   # Firebase Config
   VITE_FIREBASE_API_KEY=your_firebase_api_key
   VITE_FIREBASE_PROJECT_ID=your_firebase_project_id
   VITE_FIREBASE_DATABASE_URL=your_firebase_database_url
   ```
5. Run the development server: `npm run dev`

## Technical Implementation

### Pixel Board
The pixel board uses Firebase Realtime Database for real-time collaboration:
- Canvas data is stored as pixel coordinates and RGB values
- Real-time updates using Firebase listeners
- Automatic synchronization across all connected users
- Optimized data structure for performance
- Transaction-based updates ensure data consistency

## Prerequisites

- Node.js (v14 or later)
- npm or yarn
- MetaMask wallet
- Firebase account

## Project Structure

- `src/App.vue` - Main application component
- `src/components/`
  - `Balance.vue` - Displays GALA balance
  - `BurnGala.vue` - Handles token burning
  - `TransferGala.vue` - Handles token transfers
  - `PixelBoard.vue` - Interactive community canvas
  - `WalletConnect.vue` - Handles wallet connection
- Environment variables are defined in `.env`
- Vite configuration in `vite.config.ts`

## Usage

1. Open your browser and navigate to `http://localhost:3000`
2. Click "Connect Wallet" to connect your MetaMask wallet
3. Once connected, you'll see your GALA balance
4. Navigate to the Pixel Board tab to start drawing
5. Each pixel costs 1 GALA to place
6. Click "Claim Pixels" to burn GALA and save your changes

## Development

The application is built with:
- Vue 3 (Composition API)
- TypeScript
- Vite
- Firebase Realtime Database
- GalaChain Connect library

## Additional Resources

- [GalaChain Documentation](https://docs.galachain.com)
- [GalaChain Connect Library](https://github.com/GalaChain/sdk)
- [Firebase Documentation](https://firebase.google.com/docs)

