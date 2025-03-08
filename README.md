# SAS.6-NAME

<table style="width: 100%;">
  <tr>
    <td>
      <img src="https://github.com/BorneLabs/SAS.6-NAME/blob/main/Assets/Media/Images/NAME-1.jpg.gif" alt="NAME GIF" style="width:100%; height:auto;">
    </td>
  </tr>
</table>



  # [ Neural Agent Modelling Engine ] Mobile App Prototype Documentation

## 1. Introduction

We are Bornelabs, and we are developing NAME—a mobile app that lets users create their own AI agents by uploading documents and chatting with the agent they’ve named. For our first prototype, we’ve chosen a hybrid approach. This means we’re using a Kodular-built app for the user interface and a cloud-based backend to handle the heavy AI processing using a Retrieval-Augmented Generation (RAG) method.

Our project’s purpose is to validate the idea quickly by offloading complex AI processing to the backend while keeping the mobile interface simple and user-friendly. We plan to enhance the app further with fine-tuning capabilities later on.

## 2. Requirements

### Functional Requirements

- **User Interface:**  
  We will create a Kodular-based mobile app that allows users to:
  - Upload documents.
  - Name their AI agent.
  - Initiate and maintain a chat with the AI agent.

- **Backend Processing:**  
  We will set up a cloud-based backend that:
  - Receives document uploads from the mobile app.
  - Converts documents into vector embeddings for storage and retrieval.
  - Implements RAG to fetch and combine relevant information from the uploaded documents with the AI’s responses.
  - Returns the generated responses back to the mobile app.

- **Data Flow:**  
  The app must securely send user inputs and document data to the backend and display the AI’s responses promptly.

### Non-Functional Requirements

- **Performance:**  
  We want the app to respond quickly, with minimal latency between sending a query and receiving a response.

- **Scalability:**  
  The backend should handle multiple users and large document uploads without performance degradation.

- **Security:**  
  Data transmitted between the mobile app and the backend must be encrypted and securely stored.

- **User Experience:**  
  We will design the app to be clean and intuitive so that users of all technical levels can easily navigate it.

- **Maintainability:**  
  Our codebase—both in Kodular and on the backend—will be structured clearly to allow for easy updates as we add fine-tuning and other features later.
