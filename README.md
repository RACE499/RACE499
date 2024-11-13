import tkinter as tk
from tkinter import messagebox
from collections import deque
import mariadb
from mariadb import Error

class CounterApp:
    def __init__(self, master):
        self.master = master
        master.title("Enrolling")  # Renamed GUI window

        # Initialize queues
        self.counter1_queue = deque()
        self.counter2_queue = deque()

        # Global ticket counter
        self.ticket_counter = 1

        # Connect to MariaDB
        self.connect_to_db()

        # Create counter widgets
        self.create_counter_widgets(master, 1)
        self.create_counter_widgets(master, 2)

        # Status label
        self.status_label = tk.Label(master, text="No customers in queue", font=("Arial", 14))
        self.status_label.pack()

        # Initialize the database with required entries
        self.initialize_db()

    def connect_to_db(self):
        try:
            self.connection = mariadb.connect(
                host='localhost',  # Adjust if your MariaDB is hosted elsewhere
                port=3306,         # Specify the port for MariaDB
                database='queueing',  # Replace with your database name
                user='root',       # Using root user
                password='12345'   # Hard-coded password
            )
            print("Connected to MariaDB")
        except Error as e:
            messagebox.showerror("Connection Error", f"Error: {e}")
            self.master.quit()

    def initialize_db(self):
        try:
            cursor = self.connection.cursor()
            cursor.execute("""
                CREATE TABLE IF NOT EXISTS enrollment (
                    Counters INT PRIMARY KEY,
                    Tickets INT
                )
            """)
            # Insert initial records if they do not exist
            cursor.execute("INSERT INTO enrollment (Counters, Tickets) VALUES (1, 0) ON DUPLICATE KEY UPDATE Tickets = Tickets;")
            cursor.execute("INSERT INTO enrollment (Counters, Tickets) VALUES (2, 0) ON DUPLICATE KEY UPDATE Tickets = Tickets;")
            self.connection.commit()
            print("Database initialized with counters.")
        except Error as e:
            print(f"Error initializing database: {e}")

    def create_counter_widgets(self, master, counter_number):
        label = tk.Label(master, text=f"Counter {counter_number} Queue:", font=("Arial", 16))
        label.pack()

        self.queue_label = tk.Label(master, text="Queue: []", font=("Arial", 14))
        self.queue_label.pack()

        # Enlarged buttons with larger fonts
        button_width = 30  # Set the button width
        button_height = 3   # Set the button height (number of text lines)
        button_font = ("Arial", 14)  # Set the button font size

        add_button = tk.Button(
            master,
            text=f"Add Customer to Counter {counter_number}",
            command=lambda: self.add_customer(counter_number),
            width=button_width,
            height=button_height,
            font=button_font
        )
        add_button.pack(pady=10)

        serve_button = tk.Button(
            master,
            text=f"Serve Customer at Counter {counter_number}",
            command=lambda: self.serve_customer(counter_number),
            width=button_width,
            height=button_height,
            font=button_font
        )
        serve_button.pack(pady=10)

    def add_customer(self, counter_number):
        # Add a customer to the queue and update the ticket count
        ticket_number = self.ticket_counter  # Get the current ticket number
        self.ticket_counter += 1  # Increment the ticket counter

        if counter_number == 1:
            self.counter1_queue.append(ticket_number)
            self.update_queue_label(1)
            self.update_ticket_in_db(1, ticket_number)  # Update ticket count for Counter 1
        else:
            self.counter2_queue.append(ticket_number)
            self.update_queue_label(2)
            self.update_ticket_in_db(2, ticket_number)  # Update ticket count for Counter 2

    def update_ticket_in_db(self, counter_number, ticket_number):
        # Update the ticket count in the database for the specified counter
        try:
            cursor = self.connection.cursor()
            cursor.execute("UPDATE enrollment SET Tickets = ? WHERE Counters = ?", (ticket_number, counter_number))
            self.connection.commit()
            print(f"Counter {counter_number} updated with Ticket: {ticket_number} in database.")
        except Error as e:
            print(f"Error updating Ticket: {e}")

    def serve_customer(self, counter_number):
        # Serve a customer and update the queue and ticket count
        if counter_number == 1 and self.counter1_queue:
            served_customer = self.counter1_queue.popleft()
            print(f"Counter 1 served customer: Ticket {served_customer}")
            self.update_queue_label(1)
            self.increment_ticket_in_db(1)  # Increment the ticket count in the database after serving
        elif counter_number == 2 and self.counter2_queue:
            served_customer = self.counter2_queue.popleft()
            print(f"Counter 2 served customer: Ticket {served_customer}")
            self.update_queue_label(2)
            self.increment_ticket_in_db(2)  # Increment the ticket count in the database after serving

    def increment_ticket_in_db(self, counter_number):
        # Increment the ticket count in the database for the specified counter
        try:
            cursor = self.connection.cursor()
            cursor.execute("UPDATE enrollment SET Tickets = Tickets + 1 WHERE Counters = ?", (counter_number,))
            self.connection.commit()
            print(f"Counter {counter_number} ticket count incremented in database.")
        except Error as e:
            print(f"Error incrementing Ticket: {e}")

    def update_queue_label(self, counter_number):
        # Update the display of the queue
        if counter_number == 1:
            self.queue_label.config(text=f"Queue: {list(self.counter1_queue)}")
        else:
            self.queue_label.config(text=f"Queue: {list(self.counter2_queue)}")

    def close_db_connection(self):
        # Close the database connection
        if hasattr(self, 'connection') and self.connection:
            self.connection.close()
            print("MariaDB connection closed.")

if __name__ == "__main__":
    root = tk.Tk()
    counter_app = CounterApp(root)

    # Handle closing the main window
    root.protocol("WM_DELETE_WINDOW", counter_app.close_db_connection)
    root.mainloop()
