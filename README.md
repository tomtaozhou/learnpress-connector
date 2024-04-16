# LearnPress Connector
<?php
/**
 * Plugin Name: LearnPress Connector Plugin
 * Description: Automatically logs LearnPress quiz activity start and end times, and scores into WordPress posts.
 * Version: 1.0
 * Author: Tao Zhou
 */

// Register a custom post type for quiz activity logs
function register_quiz_activity_post_type() {
    register_post_type('quiz_activity', [
        'labels' => [
            'name' => 'Quiz Activities',
            'singular_name' => 'Quiz Activity'
        ],
        'public' => true,
        'has_archive' => true,
        'supports' => ['title', 'editor'],
    ]);
}
add_action('init', 'register_quiz_activity_post_type');

// Capture start time when a quiz starts
function handle_quiz_start($quiz_id, $user_id, $course_id) {
    $start_time = current_time('mysql');
    update_user_meta($user_id, "start_time_{$quiz_id}", $start_time);
}

// Capture end time and score when a quiz finishes
function handle_quiz_finish($quiz_data, $quiz_id, $user_id) {
    $end_time = current_time('mysql');
    $start_time = get_user_meta($user_id, "start_time_{$quiz_id}", true);
    $score = $quiz_data->get_user_quiz_grade($user_id, $quiz_id);
    
    // Create a new WordPress post
    $post_id = wp_insert_post([
        'post_type' => 'quiz_activity',
        'post_title' => "Quiz {$quiz_id} Activity for User {$user_id}",
        'post_content' => "Quiz started at: {$start_time}, ended at: {$end_time}. Score: {$score}",
        'post_status' => 'publish',
        'meta_input' => [
            'quiz_id' => $quiz_id,
            'user_id' => $user_id,
            'course_id' => $course_id,
            'start_time' => $start_time,
            'end_time' => $end_time,
            'score' => $score
        ]
    ]);

    if (is_wp_error($post_id)) {
        error_log('Failed to create post: ' . $post_id->get_error_message());
    }
}

add_action('learn_press_user_starts_quiz', 'handle_quiz_start', 10, 3);
add_action('learn_press_quiz_completed', 'handle_quiz_finish', 10, 3);
