## Example Casino Reroute

Example of how to reroute game connection towards yourself first instead of directly to casino game provisioner.
  
  In this case using demo games from Bgaming.com as example, utilizing the base scaffold of [Laravel](https://github.com/laravel/laravel).

                                                                                                      
                                                                                                      
                                                                                                      
                                                                                                      
                                                                                                    ```
<?
  Route::get('/entryBgaming', function (Request $request) {

    if(!$request->player) {
      return 'You did not specify player id in your request';
    }

    $player = $request->player;
          //Normally first retrieve the gamesession, in bgaming in the form of "playtoken" and locally assign this to the specific player-id.     
    $http = Http::get('http://bgaming-network.com/play/FruitMillion/FUN?server=demo');

    // Edits to make player contact us instead
    // Incase of stake.com, softswiss, blueocean etc. this done in so many different ways, for example can also be done through google-analytics script (so looks like a legitimate upstream call from player side to google) but can also be in form of recaptcha etc or in this case we're changing the api url
          $replaceAPItoOurs = str_replace('https://bgaming-network.com/api/', 'http://vps-70325c4a.vps.ovh.net/api/bgaming/', $http);

          // Remove existing analytics, you can also replace by your own newrelic ID
          $removeExistingAnalytics = str_replace('https://boost.bgaming-network.com/analytics.js', ' ', $replaceAPItoOurs);
          $removeExistingAnalytics = str_replace('https://www.googletagmanager.com/gtag/js?id=UA-98852510-1', '', $removeExistingAnalytics);
          $removeExistingAnalytics = str_replace('98852510', ' ', $removeExistingAnalytics);
          $removeExistingAnalytics = str_replace('sentry', ' ', $removeExistingAnalytics);
          $removeExistingAnalytics = str_replace('https://js-agent.newrelic.com/nr-1215.min.js', ' ', $removeExistingAnalytics);

          $finalGameContent = $removeExistingAnalytics;

    return $finalGameContent;
  });

  // In our case we're receiving on similar link:
  // http://vps-70325c4a.vps.ovh.net/api/bgaming/FruitMillion/276392/0c9be53e-ddee-4fe6-a28f-1125c6db1e0d
  //
  // We forward it to: (feigning real player action)
  // https://bgaming-network.com/api/FruitMillion/276392/cace7dfb-608d-465d-8dc7-28a929cc7bbb
  Route::any('/bgaming/FruitMillion/{currencyCode}/{playToken}', function ($currencyCode, $playToken, Request $request) {

    $savedRequest = $request->getContent();
    $finalLink = 'https://bgaming-network.com/api/FruitMillion/'.$currencyCode.'/'.$playToken;

    // We imitate being the player towards Bgaming:
          $liftOff = Http::retry(1, 1000, function ($exception, $request) {
                  return $exception instanceof ConnectionException;
          })->withBody(
              $savedRequest, 'application/json'
          )->post($finalLink);

    // We alter the response from legitimate Bgaming (or whatever provider):
          $data_origin = json_decode($liftOff, true);
    $data_origin['options']['currency']['code'] = 'COCO$';

    // Here you have to catch the BET & WIN amount. You then send a gameRequest to the casino, with the amount player is trying to bet, if player has enough then spin can commence
    // the balance modification then is done on the casino level. The casino industry standard is simply to send game results and then leave responsibillity to the casino to do right balance modification to player.
    $data_origin['balance']['wallet'] = '28200';

    // Let's say we want to "respin" the game (because of too high win), simply just repeat the "$savedRequest" (spin request by player) to the game server.
    // That way player will never even notice or see the missed win.


   return response()->json($data_origin);
});```
