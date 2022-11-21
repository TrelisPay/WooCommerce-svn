<?php
/**
 * @link              https://www.Trelis.com
 * @since             1.0.16
 * @package           Trelis_Crypto_Payments
 *
 * @wordpress-plugin
 * Plugin Name:       Trelis Crypto Payments
 * Plugin URI:        https://docs.trelis.com/products/woocommerce-plugin
 * Description:       Accept USDC or Ether payments directly to your wallet. Your customers pay by connecting any Ethereum wallet. No Trelis fees!
 * Version:           1.0.16
 * Requires at least: 6.1
 * Requires PHP:      7.4
 * Author:            Trelis
 * Author URI:        https://www.Trelis.com
 * License:           GPL-3.0
 * License URI:       http://www.gnu.org/licenses/gpl-3.0.txt
 * Text Domain:       trelis-crypto-payments
 * Domain Path:       /languages
 */




add_filter( 'woocommerce_currencies', 'trelis_add_crypto' );

function trelis_add_crypto( $currencies ) {
    $currencies['ETH'] = __( 'ETH', 'woocommerce' );
    $currencies['USDC'] = __( 'USDC', 'woocommerce' );
    return $currencies;
}

add_filter('woocommerce_currency_symbol', 'trelis_add_currency_symbols', 10, 2);

function trelis_add_currency_symbols( $currency_symbol, $currency ) {
    switch( $currency ) {
        case 'ETH': $currency_symbol = 'ETH'; break;
        case 'USDC': $currency_symbol = 'USDC'; break;
    }
    return $currency_symbol;
}

function trelis_get_currency() {
    global  $woocommerce;
    $currency = get_woocommerce_currency();

    switch ($currency) {
        case 'USD':
            return "USDC";
        case 'USDC':
        case 'ETH':
            return $currency;
        default:
            return null;
    }
}


/*
* Payment callback Webhook, Used to process the payment callback from the payment gateway
*/

if (!defined('ABSPATH')) exit;
function trelis_payment_confirmation_callback()
{
    $trelis = WC()->payment_gateways->payment_gateways()['trelis'];
    $json = file_get_contents('php://input');

    $expected_signature = hash_hmac('sha256', $json,  $trelis->get_option('webhook_secret'));
    if ( $expected_signature != $_SERVER["HTTP_SIGNATURE"])
        return "Failed";

    $data = json_decode($json);

    $orders = get_posts( array(
        'post_type' => 'shop_order',
        'posts_per_page' => -1,
        'post_status' => 'any',
        'meta_key'   => '_transaction_id',
        'meta_value' => json_decode(json_encode($data->mechantProductKey)),
    ));

    if (empty($orders))
        return "Failed";

    $order_id = $orders[0]->ID;
    $order = wc_get_order($order_id);

    if ($order->get_status() == 'processing' || $order->get_status() == 'complete')
        return "Already processed";

    if ($data->isSuccessful === false) {
        $order->add_order_note('Trelis payment failed!<br>Body: '.$json);
        $order->add_order_note("Expected amount " . $data->requiredPaymentAmount . ", received " . $data->paidAmount, true);
        $order->save();
        return "Failed";
    }

    $order->add_order_note("Payment complete!", true);
    $order->payment_complete();
    $order->reduce_order_stock();
    return "Processed";
}

add_action("rest_api_init", function () {
    register_rest_route(
        'trelis/v3',
        '/payment',
        array(
            'methods' => 'POST',
            'callback' => 'trelis_payment_confirmation_callback',
            'permission_callback' => '__return_true'
        ),
    );
});



if (in_array('woocommerce/woocommerce.php', apply_filters('active_plugins', get_option('active_plugins')))) {

    add_filter('woocommerce_payment_gateways', 'trelis_add_gateway_class');
    function trelis_add_gateway_class($gateways)
    {
        $gateways[] = 'WC_Trelis_Gateway';
        return $gateways;
    }

    add_action('plugins_loaded', 'trelis_init_gateway_class');
    function trelis_init_gateway_class()
    {
        if (!class_exists('WC_Payment_Gateway'))
            return; // if the WC payment gateway class is not available, do nothing
        class WC_Trelis_Gateway extends WC_Payment_Gateway
        {

            public function __construct()
            {
                $this->id = 'trelis';
                $this->icon = 'https://www.trelis.com/assets/trelis.2e0ed160.png';
                $this->supports = array(
                    'products'
                );

                $this->trelis_init_form_fields();
                $this->init_settings();
                $this->title = "Trelis Crypto Payments";
                $this->enabled = $this->get_option('enabled');
                $this->api_key = $this->get_option('api_key');
                $this->api_secret = $this->get_option('api_secret');
                add_action('woocommerce_update_options_payment_gateways_' . $this->id, array($this, 'process_admin_options'));

                if (is_checkout()) {
                    wp_register_style("trelis", plugins_url('/assets/css/trelis.css', __FILE__), '', '1.0.0');
                    wp_enqueue_style('trelis');
                }
            }

            public function trelis_init_form_fields()
            {
                $this->form_fields = array(
                    'enabled' => array(
                        'title' => 'Enable/Disable',
                        'label' => 'Enable Trelis Pay',
                        'type' => 'checkbox',
                        'description' => '',
                        'default' => 'yes'
                    ),
                    'api_url' => array(
                        'title' => 'API Webhook URL',
                        'type' => 'text',
                        'custom_attributes' => array('readonly' => 'readonly'),
                        'default' => home_url()."/wp-json/trelis/v3/payment"
                    ),
                    'api_key' => array(
                        'title' => 'API Key',
                        'type' => 'text'
                    ),
                    'api_secret' => array(
                        'title' => 'API Secret',
                        'type' => 'password'
                    ),
                    'webhook_secret' => array(
                        'title' => 'Webhook Secret',
                        'type' => 'password'
                    ),
                );
            }

            public function process_payment($order_id)
            {
                global $woocommerce;
                $order = wc_get_order($order_id);


                $currency = trelis_get_currency();

                if (!$currency) {
                    wc_add_notice("Trelis doesn't support that currency currently.", 'error');
                    return;
                }


                $apiKey = $this->get_option('api_key');
                $apiSecret = $this->get_option('api_secret');
                $apiUrl = 'https://api.trelis.com/dev-api/create-dynamic-link?apiKey=' . $apiKey . "&apiSecret=" . $apiSecret;

                $args = array(
                    'headers' => array(
                        'Content-Type' => "application/json"
                    ),
                    'body' => json_encode(array(
                        'productName' => get_bloginfo( 'name' ),
                        'productPrice' => $order->total,
                        'currencyType' => $currency,
                        'redirectLink' => $this->get_return_url($order)
                    ))
                );

                $response = wp_remote_post($apiUrl, $args);

                if (!is_wp_error($response)) {
                    $body = json_decode($response['body'], true);

                    if ($body["message"] == 'Successfully created product') {
                        $order->add_order_note($response['body'], false);
                        $str = explode("/", $body["data"]["productLink"]);
                        $paymentID = $str[count($str)-1];
                        $order->set_transaction_id($paymentID);
                        $order->save();
                        $woocommerce->cart->empty_cart();

                        return array(
                            'result' => 'success',
                            'redirect' => $body["data"]["productLink"],
                        );
                    } else {
                        wc_add_notice($body["error"], 'error');
                        return;
                    }
                } else {
                    wc_add_notice($response->get_error_message(), 'error');
                    wc_add_notice('Connection error.', 'error');
                    return;
                }
            }
        }
    }
}

