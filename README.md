# emacs_couch_db_access
Use ```M-x couch-list-docs``` after changing ```couch-username``` and ```couch-password``` in this lisp-file: 

```
;;; couchdb-utils.el --- CouchDB utilities -*- lexical-binding: t -*-

;; Your CouchDB functions here...

(require 'url)
(require 'json)

(defvar couch-base-url "http://127.0.0.1:5984")
(defvar couch-username "admin")
(defvar couch-password "1234")

(defun couch-auth-string ()
  "Create basic auth string for CouchDB."
  (concat "Basic "
          (base64-encode-string
           (concat couch-username ":" couch-password))))


(defun couch-display-doc (db-name doc-id)
  "Display the content of DOC-ID from DB-NAME with nice formatting."
  (let ((url-request-method "GET")
        (url-request-extra-headers
         `(("Authorization" . ,(couch-auth-string))
           ("Accept" . "application/json")))
        (buffer-name (format "*couch-doc-%s*" doc-id)))  ; Create unique buffer name
    (with-current-buffer (get-buffer-create buffer-name)
      (erase-buffer))
    (url-retrieve 
     (format "%s/%s/%s" couch-base-url db-name doc-id)
     (lambda (_status)
       (goto-char (point-min))
       (search-forward "\n\n")
       (let* ((raw-json (buffer-substring-no-properties (point) (point-max)))
              (json-data (json-read-from-string raw-json))
              (doc-buffer (get-buffer buffer-name)))
         (when doc-buffer
           (with-current-buffer doc-buffer
             (erase-buffer)
             (insert (format "Document %s from %s:\n\n" doc-id db-name))
             (let ((json-encoding-pretty-print t)
                   (json-encoding-default-indentation "  "))
               (insert (json-encode json-data)))
             (goto-char (point-min))
             ;; Use json-mode instead of js-mode
             (json-mode)
             ;; Additional formatting
             (indent-region (point-min) (point-max))
             (display-buffer doc-buffer)))))
     nil t)))


(defun couch-list-docs (db-name)
  "List all documents in DB-NAME with clickable links and additional info."
  (interactive "sDatabase name: ")
  (let ((url-request-method "GET")
        (url-request-extra-headers
         `(("Authorization" . ,(couch-auth-string))
           ("Accept" . "application/json"))))
    (with-current-buffer (get-buffer-create "*couch-docs*")
      (erase-buffer))
    (url-retrieve 
     (format "%s/%s/_all_docs?include_docs=true" couch-base-url db-name)
     (lambda (_status)
       (goto-char (point-min))
       (search-forward "\n\n")
       (let* ((json-object-type 'plist)
              (json-array-type 'list)
              (data (json-read))
              (docs-buffer (get-buffer "*couch-docs*")))
         (when docs-buffer
           (with-current-buffer docs-buffer
             (erase-buffer)
             (insert (format "Documents in %s:\n\n" db-name))
             ;; Create header
             (insert (format "%-15s  %-50s  %s\n" "ID" "Title" "Year"))
             (insert (make-string 80 ?-) "\n")
             ;; List documents
             (dolist (row (plist-get data :rows))
               (let* ((doc-id (plist-get row :id))
                     (doc (plist-get row :doc))
                     (title (or (plist-get doc :title) ""))
                     (year (or (plist-get doc :year) "")))
                 (insert-button (format "%-15s" doc-id)
                              'action (lambda (_)
                                      (couch-display-doc db-name doc-id))
                              'follow-link t
                              'help-echo "Click to view document")
                 (insert (format "  %-50s  %s\n" 
                               (truncate-string-to-width title 50)
                               year))))
             (goto-char (point-min))
             (display-buffer docs-buffer)))))
     nil t)))



(provide 'couchdb-utils)
;;; couchdb-utils.el ends here
```
