package com.mycompany.javafxapplication1;

import java.util.Optional;
import javafx.collections.ObservableList;
import javafx.event.ActionEvent;
import javafx.fxml.FXML;
import javafx.fxml.FXMLLoader;
import javafx.scene.Parent;
import javafx.scene.Scene;
import javafx.scene.control.Button;
import javafx.scene.control.ButtonType;
import javafx.scene.control.TableColumn;
import javafx.scene.control.TableView;
import javafx.scene.control.TextField;
import javafx.scene.control.TextArea; 
import javafx.scene.control.Label;
import javafx.scene.control.Alert;
import javafx.scene.control.cell.PropertyValueFactory;
import javafx.stage.Stage;

public class SecondaryController {

    @FXML
    private TableView<UserFiles> filesTableView;
    
    @FXML
    private TableColumn<UserFiles, String> fileNameCol;
    
    @FXML
    private TableColumn<UserFiles, String> fileContentCol;private User currentUser;

    @FXML
    private TextField terminalInput;

    @FXML
    private TextArea terminalOutput; 

    @FXML
    private TextField userTextField;

    @FXML
    private TableView<User> dataTableView;

    @FXML
    private Button secondaryButton;

    @FXML
    private Button refreshBtn;

    @FXML
    private Button delBtn;

    @FXML
    private TextField customTextField;

    private void refreshTable() {
        try {
            DB db = new DB();
            ObservableList<User> data = db.getDataFromTable();
            dataTableView.setItems(data);
            dataTableView.refresh();
        } catch (ClassNotFoundException e) {
            showError("Database Error", "Could not refresh table: " + e.getMessage());
        }
    }

    @FXML
    private void RefreshBtnHandler(ActionEvent event) {
        refreshTable();
        if (customTextField != null && customTextField.getScene() != null) {
            Stage stage = (Stage) customTextField.getScene().getWindow();
            Object ud = stage.getUserData();
            if (ud != null) {
                customTextField.setText(ud.toString());
            }
        }
    }

    @FXML
    private void delAction() {
        if (currentUser == null || !currentUser.isAdmin()) {
            showError("Permission Denied", "Only ADMIN can delete users.");
            return;
        }

        User selected = dataTableView.getSelectionModel().getSelectedItem();
        if (selected == null) {
            showError("Selection Error", "No user selected.");
            return;
        }
        
        if (selected.getUser().equalsIgnoreCase(currentUser.getUser())) {
            showError("Action Denied", "You cannot delete your own account while logged in.");
            return;
        }

        Alert alert = new Alert(Alert.AlertType.CONFIRMATION);
        alert.setTitle("Confirm Delete");
        alert.setHeaderText("Delete user: " + selected.getUser());
        alert.setContentText("Are you sure? This cannot be undone.");

        Optional<ButtonType> result = alert.showAndWait();
        if (result.isPresent() && result.get() == ButtonType.OK) {
            try {
                DB db = new DB();
                boolean deleted = db.deleteUser(selected.getUser());
                if (deleted) {
                    refreshTable();
                }
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }

    @FXML
    private void handleSaveFile() {
        if (currentUser == null) {
            showError("Error", "No user logged in.");
            return;
        }

        String content = customTextField.getText();
        if (content == null || content.isEmpty()) {
            showError("Input Error", "Custom data field is empty.");
            return;
        }

        String fileName = "note_" + System.currentTimeMillis() + ".txt";
        String owner = currentUser.getUser();

        DB db = new DB();
        try {
            db.saveEncryptedFile(owner, fileName, content);
            
            Alert alert = new Alert(Alert.AlertType.INFORMATION);
            alert.setTitle("Success");
            alert.setHeaderText("File Encrypted & Saved");
            alert.setContentText("Your data has been secured with AES-256 encryption.");
            alert.showAndWait();
            
            customTextField.clear();
        } catch (Exception e) {
            e.printStackTrace();
            showError("Encryption Error", "Failed to save file: " + e.getMessage());
        }
    }

    @FXML
    private void switchToPrimary() {
        try {
            Stage primaryStage = (Stage) secondaryButton.getScene().getWindow();
            FXMLLoader loader = new FXMLLoader(getClass().getResource("primary.fxml"));
            Parent root = loader.load();
            
            Scene scene = new Scene(root, 640, 480);
            Stage newStage = new Stage();
            newStage.setScene(scene);
            newStage.setTitle("Login");
            newStage.show();
            
            primaryStage.close();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    public void initialise(User loggedIn) {
        this.currentUser = loggedIn;
        
        if (userTextField != null) {
            userTextField.setText(loggedIn.getUser() + " (" + loggedIn.getRole() + ")");
        }

        if (delBtn != null) {
            delBtn.setDisable(!loggedIn.isAdmin());
        }

        if (dataTableView != null) {
            TableColumn<User, String> userCol = new TableColumn<>("Username");
            userCol.setCellValueFactory(new PropertyValueFactory<>("user"));

            TableColumn<User, String> roleCol = new TableColumn<>("Role");
            roleCol.setCellValueFactory(new PropertyValueFactory<>("role"));

            dataTableView.getColumns().setAll(userCol, roleCol);
            refreshTable();
        }

        if (fileNameCol != null && fileContentCol != null) {
            fileNameCol.setCellValueFactory(new PropertyValueFactory<>("filename"));
            fileContentCol.setCellValueFactory(new PropertyValueFactory<>("content"));
            refreshFilesTable();
        } else {
            System.out.println("WARNING: File Table Columns are null. Skipping file table setup.");
        }
    }
    

    @FXML
    private void runTerminalCommand() {
        String command = terminalInput.getText();
        if (command == null || command.isEmpty()) return;

        try {
            ProcessBuilder pb = new ProcessBuilder("/bin/bash", "-c", command);
            pb.redirectErrorStream(true); 
            Process process = pb.start();

            java.io.BufferedReader reader = new java.io.BufferedReader(
                    new java.io.InputStreamReader(process.getInputStream())
            );

            StringBuilder output = new StringBuilder();
            String line;
            while ((line = reader.readLine()) != null) {
                output.append(line).append("\n");
            }
            terminalOutput.setText(output.toString());

        } catch (java.io.IOException e) {
            terminalOutput.setText("Error: " + e.getMessage());
        }
    }

    private void refreshFilesTable() {
        DB db = new DB();
        ObservableList<UserFiles> files = db.getUserFiles(currentUser.getUser());
        filesTableView.setItems(files);
    }

    @FXML
    private void handleDecryptFile() {
        UserFiles selected = filesTableView.getSelectionModel().getSelectedItem();
        if (selected == null) {
            showError("Selection Error", "Please select an encrypted file to decrypt.");
            return;
        }

        try {
            DB db = new DB();
            String decryptedText = db.decryptFile(selected.getContent(), selected.getKey());
            
            Alert alert = new Alert(Alert.AlertType.INFORMATION);
            alert.setTitle("Decryption Successful");
            alert.setHeaderText("Original Content:");
            alert.setContentText(decryptedText);
            alert.showAndWait();
        } catch (Exception e) {
            showError("Decryption Error", "Failed to decrypt: " + e.getMessage());
        }
    }
    
    @FXML
private void handleShareAction() {
    String userToShareWith = shareUsernameField.getText();
    int fileIdToShare = 1; 
    DB db = new DB();
    db.shareFile(fileIdToShare, userToShareWith);
    terminalArea.appendText("\n[ACL] Permission granted: " + userToShareWith + " can now access File " + fileIdToShare);
}

    private void showError(String title, String content) {
        Alert alert = new Alert(Alert.AlertType.ERROR);
        alert.setTitle(title);
        alert.setContentText(content);
        alert.showAndWait();
    }
}
