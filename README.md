## It is a simple one, but you can amuse yourself by chatting with it. And if you can upgrade it, let me know. I had this problem with the autonomous inputs that I can't cut while the user starts writing, so it kind of speaks over you!


  import os
  import random
  import time
  import threading
  import Colorama
  import re
  from collections import Counter
  from colorama import Fore, Style

colorama.init(autoreset=True)

# Global variables
randomness_level = 50  # Default randomness level
autonomous_level = 10  # Default autonomous output delay in seconds
max_idle_time = 2  # Maximum idle time for spontaneous output

recent_inputs = []  # To store recent user inputs
allow_autonomous_output = True  # Flag to control autonomous output

def normalize_popularity(folder_path):
    """ Normalize popularity values in response files."""
    sen_file_path = os.path.join(folder_path, "sen.txt")
    res_file_path = os.path.join(folder_path, "res.txt")

    try:
        with open(sen_file_path, "r") as sen_file:
            sen_texts = sen_file.readlines()

        with open(res_file_path, "r") as res_file:
            responses = res_file.readlines()

        new_responses = []
       
        for response in responses:
            try:
                popularity_str, text = response.split(" ", 1)
                popularity_value = float(popularity_str[1:-1].replace('%', ''))  # Remove '<', '>', and '%'
                normalized_value = min(max(popularity_value / 100, 0.0), 1.0) * 100  # Normalize to a percentage
                new_responses.append(f"<{normalized_value:.0f}%> {text.strip()}\n")
            except ValueError:
                print(f"Invalid popularity value in response: {response.strip()}")
                continue  # Skip invalid responses

        with open(res_file_path, "w") as res_file:
            res_file.writelines(new_responses)

    except FileNotFoundError as e:
        print(f"File not found: {e}")
    except Exception as e:
        print(f"Error normalizing popularity: {e}")

def normalize_popularity_in_data(data_folder):
    """Normalize popularity for all sen.txt and res.txt files in the data folder and its subdirectories."""
    for root, dirs, files in os.walk(data_folder):
        if "sen.txt" in files and "res.txt" in files:
            print(f"Normalizing files in folder: {root}")
            normalize_popularity(root)

def save_response(folder_path, user_input, response, popularity):
    if not os.path.exists(folder_path):
        os.makedirs(folder_path)
   
    sen_file_path = os.path.join(folder_path, "sen.txt")
    res_file_path = os.path.join(folder_path, "res.txt")
   
    popularity = min(max(popularity, 0.0), 100.0)
   
    with open(sen_file_path, "a") as sen_file:
        sen_file.write(user_input + "\n")
   
    with open(res_file_path, "a") as res_file:
        res_file.write(f"<{popularity:.0f}%> {response}\n")

def clean_input(user_input):
    """Remove special characters from user input."""
    return re.sub(r'[!?.]+', '', user_input).strip()

def get_matching_folder(user_input, folder_paths):
    user_words = set(user_input.lower().split())
    best_match_score = 0
    best_match_folder = None
   
    for folder_path in folder_paths:
        sen_file_path = os.path.join(folder_path, "sen.txt")
        try:
            with open(sen_file_path, "r") as sen_file:
                sen_texts = sen_file.readlines()
                for sen_text in sen_texts:
                    sen_words = set(sen_text.lower().split())
                    match_score = len(user_words & sen_words)
                    if match_score > best_match_score:
                        best_match_score = match_score
                        best_match_folder = folder_path
        except FileNotFoundError:
            continue

    return best_match_folder

def get_combined_response(folder_path, user_input, past_inputs):
    sen_file_path = os.path.join(folder_path, "sen.txt")
    res_file_path = os.path.join(folder_path, "res.txt")

    try:
        with open(sen_file_path, "r") as sen_file:
            sen_texts = sen_file.readlines()

        with open(res_file_path, "r") as res_file:
            responses = res_file.readlines()

        user_words = set(user_input.lower().split())
        past_words = set(word for input in past_inputs for word in input.lower().split())
       
        best_match_score = 0
        best_match_indices = []
       
        for i, sen_text in enumerate(sen_texts):
            sen_words = set(sen_text.lower().split())
            match_score = 0.72 * len(user_words & sen_words) + 0.28 * len(past_words & sen_words)
            if match_score > best_match_score:
                best_match_score = match_score
                best_match_indices = [i]
            elif match_score == best_match_score:
                best_match_indices.append(i)

        words_to_use = []
        word_popularity = {}

        if best_match_indices:
            # Collect words from the best matching responses and their popularity
            for index in best_match_indices:
                popularity, response = responses[index].split(" ", 1)
                popularity_value = float(popularity[1:-1].replace('%', ''))  # Get the popularity value
                for word in response.strip().split():
                    word_popularity[word] = popularity_value  # Store word with its popularity
                    words_to_use.append(word)

        # Adjust randomness in response generation
        if randomness_level < 100:
            if randomness_level > 0:
                # Sort words by popularity and filter based on randomness level
                limit = max(1, int((100 - randomness_level) / 100 * len(words_to_use)))
                words_to_use = sorted(words_to_use, key=lambda word: word_popularity[word])[:limit]

        random.shuffle(words_to_use)
        unique_response = " ".join(words_to_use[:5])  # Take the first 5 words for brevity
        return unique_response

    except FileNotFoundError:
        return None  # Return None instead of a message
    except IndexError:
        return None  # Handle index error

def get_most_popular_words(folder_paths):
    word_counter = Counter()

    for folder_path in folder_paths:
        res_file_path = os.path.join(folder_path, "res.txt")

        try:
            with open(res_file_path, "r") as res_file:
                responses = res_file.readlines()
                for response in responses:
                    popularity_str, text = response.split(" ", 1)
                    popularity_value = float(popularity_str[1:-1].replace('%', ''))
                    if popularity_value > 0:  # Only consider popular responses
                        words = text.strip().lower().split()
                        word_counter.update(words)
        except FileNotFoundError:
            continue

    return [word for word, _ in word_counter.most_common(10)]  # Get the top 10 most common words

def generate_random_output(user_input):
    """Generate a random output based on the latest user input."""
    # Create a list of words from the user input
    words = user_input.lower().split()
    random.shuffle(words)  # Shuffle the words for randomness
    # Join the first few words to form a response
    return ' '.join(words[:5]) if words else "I'm thinking about something interesting."

def idle_comment(folder_paths, stop_event):
    """Generate random comments when user is idle."""
    while not stop_event.is_set():
        time.sleep(autonomous_level)  # Wait for the set idle time
        if allow_autonomous_output:  # Check if the user is still idle
            if recent_inputs:
                # Randomly choose one of the recent inputs
                random_input = random.choice(recent_inputs)
                # Generate a response based on the chosen input
                response = generate_random_output(random_input)
                print(f"\n{Fore.RED}[{Fore.RESET}>{Fore.RED}]: {response}")
                print(f"{Fore.GREEN}[{Fore.RESET}<{Fore.GREEN}]: ", end='')

def update_response_popularity(folder_path, user_input, new_response):
    try:
        sen_file_path = os.path.join(folder_path, "sen.txt")
        res_file_path = os.path.join(folder_path, "res.txt")

        with open(sen_file_path, "r") as sen_file:
            sen_texts = sen_file.readlines()

        with open(res_file_path, "r") as res_file:
            responses = res_file.readlines()

        user_words = set(user_input.lower().split())

        for i, sen_text in enumerate(sen_texts):
            sen_words = set(sen_text.lower().split())
            if user_words == sen_words:
                if i < len(responses):  # Check if the index exists in responses
                    popularity, _ = responses[i].split(" ", 1)
                    popularity_value = float(popularity[1:-1].replace('%', ''))  # Exclude the '<' and '>', and '%'
                    popularity_value = min(max(popularity_value, 0.0), 100.0)  # Clamp between 0 and 100
                    responses[i] = f"<{min(popularity_value + 5.0, 100.0):.0f}%> {new_response}\n"
                   
                else:
                    print(f"Error: No matching response for input at index {i}.")
                    return  # Exit if there is no response for this sentence

        with open(res_file_path, "w") as res_file:
            res_file.writelines(responses)

    except (FileNotFoundError, IndexError, ValueError) as e:
        print(f"Error updating popularity: {e}")

def clear_memory():
    folder_path = "data"
    if os.path.exists(folder_path):
        for filename in os.listdir(folder_path):
            file_path = os.path.join(folder_path, filename)
            try:
                if os.path.isfile(file_path):
                    os.remove(file_path)
                elif os.path.isdir(file_path):
                    os.rmdir(file_path)
            except Exception as e:
                print(f"Error while deleting {file_path}: {e}")
        print("Memory cleared.")
    else:
        print("No data folder found.")

def set_randomness_level(level):
    global randomness_level
    if 0 <= level <= 100:
        randomness_level = level
        print(f"Randomness level set to {randomness_level}.")
    else:
        print("Please enter a number between 0 and 100.")

def set_autonomous_level(level):
    global autonomous_level
    if 1 <= level <= max_idle_time:
        autonomous_level = level
        print(f"Autonomous output delay set to {autonomous_level} seconds.")
    else:
        print(f"Please enter a number between 1 and {max_idle_time}.")

def main():
    global recent_inputs, allow_autonomous_output

    folder_paths = []
    stop_event = threading.Event()

    data_folder = "data"
    if os.path.exists(data_folder):
        normalize_popularity_in_data(data_folder)  # Normalize all popularity values

        folder_paths = [os.path.join(data_folder, d) for d in os.listdir(data_folder) if os.path.isdir(os.path.join(data_folder, d))]

    idle_comment_thread = threading.Thread(target=idle_comment, args=(folder_paths, stop_event), daemon=True)
    idle_comment_thread.start()

    while True:
        print(f"{Fore.GREEN}[{Fore.RESET}<{Fore.GREEN}]: ", end='')
        user_input = input()

        # User has started typing
        allow_autonomous_output = False
        stop_event.set()  # Stop the idle comment thread while waiting for user input

        user_input = clean_input(user_input)  # Clean the input

        if user_input.lower() == "exit":
            break
        elif user_input.lower() == "clear memory":
            clear_memory()
            recent_inputs.clear()
            stop_event.clear()
            idle_comment_thread = threading.Thread(target=idle_comment, args=(folder_paths, stop_event), daemon=True)
            idle_comment_thread.start()
            continue
        elif user_input.lower().startswith("set autonomous level"):
            try:
                parts = user_input.split()
                if len(parts) != 4:
                    raise ValueError("Invalid command format.")
               
                level = int(parts[3])  # Convert the fourth part to an integer
                set_autonomous_level(level)
                continue
            except ValueError:
                print("Please enter a valid number for 'set autonomous level'.")
                continue
        elif user_input.lower().startswith("set random responses"):
            try:
                parts = user_input.split()
                if len(parts) != 4 or parts[2].lower() != "responses":
                    raise ValueError("Invalid command format.")
               
                level = int(parts[3])  # Convert the fourth part to an integer
                set_randomness_level(level)
                continue
            except ValueError:
                print("Please enter a valid number between 0 and 100 after 'set random responses'.")
                continue

        # Store user input
        recent_inputs.append(user_input)
       
        best_match_folder = get_matching_folder(user_input, folder_paths)
        if best_match_folder:
            response = get_combined_response(best_match_folder, user_input, recent_inputs)
            if response is not None:
                print(f"{Fore.RED}[{Fore.RESET}>{Fore.RED}]: {response}")

                # Ask for feedback
                user_feedback = input(f"{Fore.RED}Was this answer helpful? (y/n) {Fore.RESET}")
                allow_autonomous_output = True  # Re-enable after feedback

                if user_feedback.lower() == "y":
                    update_response_popularity(best_match_folder, user_input, response)
                else:
                    new_response = input(f"{Fore.RED}Teach me: {Fore.RESET}")
                    save_response(best_match_folder, user_input, new_response, 50.0)
                    update_response_popularity(best_match_folder, user_input, new_response)
            else:
                new_response = input(f"{Fore.RED}What should I answer? {Fore.RESET}")
                save_response(best_match_folder, user_input, new_response, 50.0)
        else:
            new_response = input(f"{Fore.RED}What should I say?: {Fore.RESET}")
            folder_path = f"data/{len(folder_paths)}"
            save_response(folder_path, user_input, new_response, 50.0)
            folder_paths.append(folder_path)

        stop_event.clear()  # Clear the stop event to resume idle comments
        allow_autonomous_output = True  # Re-enable after processing
     

if __name__ == "__main__":
    os.system('cls' if os.name == 'nt' else 'clear')
    print(f"{Fore.RED}========[{Fore.RESET} A L E X A N D R I A N{Fore.RED} ]========")
    print(f"{Fore.GREEN} ------< {Fore.RESET}I N T E L L I G E N C E{Fore.GREEN} >------")
    main()

