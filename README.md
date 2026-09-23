# ♻️ Educational Recycling App – Conveyor Sorter

An educational mobile game designed to help children learn how to sort waste into the correct recycling categories through an interactive conveyor-belt sorting experience.

The player must identify incoming waste items and place them into the appropriate recycling bin. The game combines simple gameplay mechanics with progressive difficulty to make learning about recycling more engaging.

## 🎯 Project Purpose

The project was developed as a **Bachelor's Degree / Licence project** focused on creating an interactive educational application for children in the field of recycling.

The main goal is to encourage environmental awareness and help children become familiar with basic waste-sorting concepts through gameplay.

## 🎮 Features

* ♻️ Waste sorting into four categories:

  * 📄 Paper
  * 🧴 Plastic
  * 🍾 Glass
  * 🌱 Organic waste
* 🚚 Animated conveyor-belt gameplay
* ⭐ Score system
* ❤️ Lives system
* 🔥 Streak tracking
* 📈 Progressive difficulty and levels
* ⏸️ Pause and restart functionality
* 🎯 Game-over state
* 💡 Recycling tips
* 📱 Responsive layout for different screen sizes and orientations
* ⚡ Animated game elements and transitions

## 🛠️ Technologies

* **React Native** – mobile application development
* **TypeScript** – typed application logic
* **Expo** – development and testing environment
* **React Native Animated API** – animations and game interactions
* **Git / GitHub** – version control and project hosting

## 📂 Project Structure

```text
Educational-recycling-app/
├── assets/
│   ├── android-icon-background.png
│   ├── android-icon-foreground.png
│   ├── android-icon-monochrome.png
│   ├── favicon.png
│   ├── icon.png
│   └── splash-icon.png
├── App.tsx
├── app.json
├── index.ts
├── package.json
├── package-lock.json
├── tsconfig.json
├── LICENSE
└── .gitignore
```

## 🚀 Getting Started

### Prerequisites

Make sure you have installed:

* [Node.js](https://nodejs.org/)
* Expo Go on your mobile device

### Installation

Clone the repository:

```bash
git clone https://github.com/Andrxw20/Educational-recycling-app.git
```

Navigate to the project directory:

```bash
cd Educational-recycling-app
```

Install the dependencies:

```bash
npm install
```

### Run the application

Start the Expo development server:

```bash
npx expo start
```

A QR code will appear in the terminal. Scan it with **Expo Go** to launch the application on a mobile device.

For the best local testing experience, make sure the computer and mobile device are connected to the same Wi-Fi network.

## 🧩 Gameplay

The game is based on a simple sorting mechanic:

1. Waste items appear on a moving conveyor belt.
2. The player identifies the type of waste.
3. The item is moved to the corresponding recycling bin.
4. Correct sorting increases the score and can build a streak.
5. Incorrect sorting results in losing a life.
6. As the game progresses, the difficulty increases through faster movement and more challenging gameplay.

## 📈 Difficulty System

The application dynamically adjusts the gameplay difficulty as the player progresses.

Difficulty can increase through:

* faster conveyor movement;
* shorter item spawn intervals;
* increased number of simultaneously active items;
* progression through game levels.

This creates a gradually more challenging experience while maintaining the basic educational objective.

## 📱 Platform

The application was developed as a cross-platform mobile application using React Native and Expo.

It has been tested using **Expo Go on a physical mobile device**.

## 🎓 Academic Project

This application was developed as part of a **Bachelor's Degree / Licence project** at the **Technical University of Moldova (UTM)**.

**Project:** Educational Application for Children in the Field of Recycling

**Author:** Andrei Oselschi

## 🤖 AI-Assisted Development

AI tools were used as development assistance during the project, including:

* ChatGPT
* Claude
* Gemini

AI assistance was used for tasks such as development support, code-related problem solving, and project documentation. The final project was tested and adapted to work in the Expo/React Native environment.

## 🔮 Possible Future Improvements

Potential future improvements include:

* additional waste categories;
* sound effects and background music;
* improved visual feedback;
* more educational content;
* multiple difficulty modes;
* achievements and rewards;
* persistent high scores;
* additional game modes;
* improved accessibility features.

## 📄 License

This project is licensed under the terms included in the [`LICENSE`](./LICENSE) file.

---

**Educational Recycling App – Conveyor Sorter**
Built with React Native, TypeScript and Expo ♻️
