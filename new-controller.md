<p><a target="_blank" href="https://app.eraser.io/workspace/McqPk0eq3RJ7X81a1d8F" id="edit-in-eraser-github-link"><img alt="Edit in Eraser" src="https://firebasestorage.googleapis.com/v0/b/second-petal-295822.appspot.com/o/images%2Fgithub%2FOpen%20in%20Eraser.svg?alt=media&amp;token=968381c8-a7e7-472a-8ed6-4a6626da5501"></a></p>



code for the plan controller



```javascript
const db = require("../models");
const constants = require("../utls/constants");
const stripeServices = require("../services/stripeServices");
const stripe = require("stripe")(process.env.STRIPE_KEY);
module.exports = {
  addPlan: async (req, res) => {
    try {
      if (req.body.name) {
        let name = req.body.name;
        req.body.name = name.toLowerCase();
      }

      let findPlan = await db.Plan.findOne({
        where: {
          isDeleted: false,
          name: req.body.name,
        },
      });

      if (findPlan) {
        return res.status(400).json({
          success: false,
          error: {
            code: 400,
            message: constants.PLAN.ALREADY_EXIST,
          },
        });
      }
      let created_product = await stripeServices.create_product({
        name: req.body.name,
      });

      if (created_product) {
        for await (const item of req.body.pricing) {
          var price = await stripe.prices.create({
            product: created_product.id,
            unit_amount: Number(item.unit_amount) * 100,
            currency: "usd",
            recurring: {
              interval: item.interval_time ? item.interval_time : "month",
              interval_count: item.interval_count ? item.interval_count : 1,
            },
          });

          item.stripe_price_id = price.id;
          req.body.stripe_price_id = item.stripe_price_id;
        }
        req.body.addedBy = req.identity.id;
        req.body.stripe_product_id = created_product.id;
        req.body.user_id = req.identity.id;
        let create_subscription_plan = await db.Plan.create(req.body);
        if (create_subscription_plan) {
          return res.status(200).json({
            success: true,
            message: constants.PLAN.CREATED,
            data: create_subscription_plan,
          });
        }
      }

      return res.status(400).json({
        success: false,
        error: {
          code: 400,
          message: constants.PLAN.UNABLE_CREATE_PLAN,
        },
      });
    } catch (err) {
      console.log(err, "===============error");
      return res.status(500).json({
        success: false,
        error: {
          code: 500,
          message: err.message || "Internal server error",
        },
      });
    }
  },
  editPlan: async (req, res) => {
    try {
      let { id } = req.body;
      if (req.body.name) {
        let name = req.body.name;
        req.body.name = name.toLowerCase();
      }
      req.body.updatedAt = new Date();
      await db.Plan.update(req.body, {
        where: { id: id },
      });

      return res.status(200).json({
        success: true,
        message: constants.PLAN.UPDATED,
      });
    } catch (err) {
      console.log(err, "===============error");
      return res.status(500).json({
        success: false,
        error: {
          code: 500,
          message: err.message || "Internal server error",
        },
      });
    }
  },
  listing: async (req, res) => {
    try {
      const { search, page, count, sortBy, status, type } = req.query;

      let where = { isDeleted: false };
      let order = [];

      if (search) {
        where = {
          ...where,
          name: { [db.Sequelize.Op.iLike]: `%${search}%` },
        };
      }

      if (sortBy) {
        const [field, sortType] = sortBy.split(" ");
        order.push([
          field || "createdAt",
          sortType === "desc" ? "DESC" : "ASC",
        ]);
      } else {
        order.push(["createdAt", "DESC"]);
      }

      if (status) {
        where.status = status;
      }
      if (type) {
        where.type = type;
      }

      let options = {
        where,
        order,
        include: [
          {
            model: db.Feature,
            as: "features",
          },
        ],
      };

      if (page && count) {
        options.offset = (Number(page) - 1) * Number(count);
        options.limit = Number(count);
      }

      const { count: total, rows: result } =
        await db.Plan.findAndCountAll(options);

      return res.status(200).json({
        success: true,
        data: result,
        total: total,
      });
    } catch (err) {
      return res.status(500).json({
        success: false,
        error: {
          code: 500,
          message: err.message || "Internal server error",
        },
      });
    }
  },
  getPlanById: async (req, res) => {
    try {
      let { id } = req.query;

      if (!id) {
        return res.status(400).json({
          success: false,
          error: {
            code: 400,
            message: constants.PLAN.ID_REQUIRED,
          },
        });
      }

      const detail = await db.Plan.findOne({
        where: {
          id: id,
          isDeleted: false,
        },
        include: [
          {
            model: db.Feature,
            as: "features",
          },
        ],
      });

      if (!detail) {
        return res.status(404).json({
          success: false,
          error: {
            code: 404,
            message: constants.PLAN.NOT_FOUND,
          },
        });
      }

      return res.status(200).json({
        success: true,
        data: detail,
      });
    } catch (err) {
      return res.status(500).json({
        success: false,
        error: {
          code: 500,
          message: err.message || "Internal server error",
        },
      });
    }
  },
  deletePlan: async (req, res) => {
    try {
      const { id } = req.query;

      if (!id) {
        return res.status(400).json({
          success: false,
          message: constants.PLAN.ID_REQUIRED,
        });
      }
      const planData = await db.Plan.findOne({
        where: {
          id: id,
          isDeleted: false,
        },
      });
      if (!planData) {
        return res.status(404).json({
          success: false,
          message: constants.PLAN.NOT_FOUND,
        });
      }

      await db.Plan.update({ isDeleted: true }, { where: { id: id } });

      return res.status(200).json({
        success: true,
        message: constants.PLAN.DELETED,
      });
    } catch (err) {
      return res.status(500).json({
        success: false,
        error: {
          code: 500,
          message: err.message || "Internal server error",
        },
      });
    }
  },
  changeStatus: async (req, res) => {
    try {
      const { id, status } = req.body;

      if (!id) {
        return res.status(400).json({
          success: false,
          error: {
            code: 400,
            message: constants.PLAN.ID_REQUIRED,
          },
        });
      }

      if (status === undefined || status === null) {
        return res.status(400).json({
          success: false,
          error: {
            code: 400,
            message: constants.PLAN.STATUS_REQUIRED,
          },
        });
      }

      const planData = await db.Plan.findOne({
        where: { id: id },
      });
      if (!planData) {
        return res.status(404).json({
          success: false,
          error: {
            code: 404,
            message: constants.PLAN.NOT_FOUND,
          },
        });
      }

      await db.Plan.update({ status }, { where: { id: id } });

      return res.status(200).json({
        success: true,
        message: constants.PLAN.STATUS_CHANGED,
      });
    } catch (err) {
      return res.status(500).json({
        success: false,
        error: {
          code: 500,
          message: err.message || "Internal server error",
        },
      });
    }
  },
};
```




code for the Feature controller

```javascript
"use strict";

const db = require("../models");
const constants = require("../utls/constants");
const { Op } = require("sequelize");

module.exports = {
  /**Add multiple features */
  addMultipleFeatures: async (req, res) => {
    try {
      const features = req.body.features;

      if (!features || !Array.isArray(features)) {
        return res.status(400).json({
          success: false,
          error: {
            code: 400,
            message: constants.onBoarding.PAYLOAD_MISSING,
          },
        });
      }

      const createdFeatures = [];

      for (const feature of features) {
        feature.createdAt = new Date();
        feature.updatedAt = new Date();
        feature.isDeleted = false;

        const isExist = await db.Feature.findOne({
          where: {
            name: feature.name,
            isDeleted: false,
          },
        });

        if (isExist) {
          continue;
        }

        try {
          const createdFeature = await db.Feature.create(feature);
          createdFeatures.push(createdFeature);
        } catch (err) {
          console.log(err);
        }
      }

      return res.status(200).json({
        success: true,
        message:
          createdFeatures.length > 0
            ? constants.FEATURE.CREATED
            : constants.FEATURE.NO_NEW_FEATURES,
        data: createdFeatures,
      });
    } catch (err) {
      console.log(err);
      return res.status(400).json({
        success: false,
        error: {
          code: 400,
          message: "Error: " + err.message,
        },
      });
    }
  },
  /** For update feature */
  updateFeature: async (req, res) => {
    try {
      let id = req.body.id;
      if (!id) {
        return res.status(400).json({
          success: false,
          error: {
            code: 400,
            message: constants.onBoarding.PAYLOAD_MISSING,
          },
        });
      }

      const Feature = await db.Feature.findOne({
        where: { id: id },
      });

      if (!Feature) {
        return res.status(404).json({
          success: false,
          error: {
            code: "404",
            message: constants.FEATURE.NOT_FOUND,
          },
        });
      }
      const data = req.body;

      const updatedFeature = await db.Feature.update(data, {
        where: { id: id },
      });

      return res.status(200).json({
        success: true,
        message: constants.FEATURE.UPDATED,
      });
    } catch (err) {
      return res.status(400).json({
        success: false,
        error: {
          code: 400,
          message: "" + err,
        },
      });
    }
  },
  activateDeactivateFeature: async (req, res) => {
    try {
      let id = req.body.id;
      if (!id) {
        return res.status(400).json({
          success: false,
          error: {
            code: 400,
            message: constants.onBoarding.PAYLOAD_MISSING,
          },
        });
      }
      const Feature = await db.Feature.findOne({
        where: { id: id },
      });
      if (!Feature) {
        return res.status(404).json({
          success: false,
          message: constants.FEATURE.NOT_FOUND,
        });
      }
      const query = {
        status: "active",
      };
      let message;
      let message1;
      if (Feature.status == "active") {
        query.status = "deactive";
        message = constants.FEATURE.STATUS_CHANGED;
      } else {
        query.status = "active";
        message1 = constants.FEATURE.STATUS_CHANGED;
      }
      const updateData = await db.Feature.update(query, {
        where: { id: id },
      });
      if (updateData) {
        return res.status(200).json({
          success: true,
          message: Feature.status === "deactive" ? message1 : message,
        });
      }
    } catch (err) {
      return res.status(400).json({
        success: false,
        error: {
          code: 400,
          message: "" + err,
        },
      });
    }
  },
  /** for get list of Feature */
  getAllFeaturesList: async (req, res) => {
    try {
      let { search, sortBy, page, count, status } = req.query;
      let where = {};
      let order = [];

      if (search) {
        where.name = {
          [Op.iLike]: `%${search}%`,
        };
      }

      where.isDeleted = false;

      if (sortBy) {
        let [field, sortType] = sortBy.split(" ");
        order.push([
          field || "createdAt",
          sortType === "desc" ? "DESC" : "ASC",
        ]);
      } else {
        order.push(["createdAt", "DESC"]);
      }

      if (status) {
        where.status = status;
      }

      const options = {
        attributes: ["status", "name", "createdAt", "updatedAt", "isDeleted"],
        where,
        order,
      };

      if (page && count) {
        options.offset = (Number(page) - 1) * Number(count);
        options.limit = Number(count);
      }

      const { rows, count: total } = await db.Feature.findAndCountAll(options);

      return res.status(200).json({
        success: true,
        data: rows,
        total: total,
      });
    } catch (err) {
      return res.status(500).json({
        success: false,
        error: {
          code: 400,
          message: "" + err,
        },
      });
    }
  },
  /** for delete Feature */
  deleteFeature: async (req, res) => {
    try {
      let id = req.query.id;
      if (!id) {
        return res.status(400).json({
          success: false,
          error: {
            code: 400,
            message: constants.onBoarding.PAYLOAD_MISSING,
          },
        });
      }
      const Feature = await db.Feature.findOne({
        where: { id: id },
      });
      if (!Feature) {
        return res.status(404).json({
          success: false,
          error: {
            code: "404",
            message: constants.FEATURE.NOT_FOUND,
          },
        });
      }

      const updateFeature = await db.Feature.update(
        { isDeleted: true },
        { where: { id: id } },
      );

      return res.status(200).json({
        success: true,
        message: constants.FEATURE.DELETED,
      });
    } catch (err) {
      return res.status(400).json({
        success: false,
        error: {
          code: 400,
          message: "" + err,
        },
      });
    }
  },
  /** for get Fetaure Post detail */
  getFeatureDetail: async (req, res) => {
    try {
      const id = req.query.id;
      if (!id) {
        return res.status(401).json({
          success: false,
          error: {
            code: 401,
            message: constants.onBoarding.PAYLOAD_MISSING,
          },
        });
      }
      const getFeature = await db.Feature.findByPk(id);
      if (getFeature) {
        return res.status(200).json({
          success: true,
          message: constants.FEATURE.RETRIEVED,
          payload: getFeature,
        });
      } else {
        return res.status(404).json({
          success: false,
          error: {
            code: 404,
            message: constants.FEATURE.NOT_FOUND,
          },
        });
      }
    } catch (err) {
      return res.status(400).json({
        success: false,
        error: {
          code: 400,
          message: "" + err,
        },
      });
    }
  },
 
};
```


Code for the subscription controller

```
"use-strict";

const db = require("../models");
const { Op } = require("sequelize");
const stripeServices = require("../services/stripeServices");
const constants = require("../utls/constants");

module.exports = {
  webhook: async (request, response) => {
    try {
      const event = request.body;
      const eventObject = event.data.object;

      switch (event.type) {
        case "customer.subscription.updated":
          const matchedPlan = await db.Plan.findOne({
            where: {
              "$pricing.stripe_price_id$": eventObject.plan.id,
            },
          });

          if (!matchedPlan) {
            return response.status(404).send("Plan not found");
          }

          const planId = matchedPlan.id;
          await db.User.update(
            { activePlan: true, activePlanId: eventObject.metadata.plan_id },
            { where: { id: eventObject.metadata.user_id } },
          );

          const updateObj = {
            valid_upto: new Date(eventObject.current_period_end * 1000),
            status: "active",
            amount: eventObject.plan.amount,
            interval_count: eventObject.metadata.interval_count,
            subscription_plan_id: planId,
          };

          if (eventObject.status) {
            await db.Subscription.update(updateObj, {
              where: { stripe_subscription_id: eventObject.id },
            });
          }
          break;

        case "invoice.payment_succeeded":
          if (eventObject.subscription) {
            const getSubscriptionData = await db.Subscription.findOne({
              where: { stripe_subscription_id: eventObject.subscription },
            });

            if (getSubscriptionData) {
              const getSubscriptionPlan = await db.Plan.findByPk(
                getSubscriptionData.subscription_plan_id,
              );

              const transactionPayload = {
                user_id: getSubscriptionData.user_id,
                paid_to: getSubscriptionPlan
                  ? getSubscriptionPlan.addedBy
                  : null,
                transaction_type: "buy_subscription",
                paymentMethod: "Stripe",
                subscription_plan_id: getSubscriptionPlan.id,
                payment_intent_id: eventObject.id,
                amount: eventObject.total || 0,
                transaction_status:
                  eventObject.status === "paid" ? "successful" : "failed",
              };

              await db.Transaction.create(transactionPayload);
            }
          }
          break;

        case "customer.subscription.trial_will_end":
          if (eventObject.id) {
            await db.Subscription.update(
              {
                trial_period_end_date: null,
                valid_upto: new Date(eventObject.current_period_end * 1000),
                trial_Status: eventObject.status,
              },
              { where: { stripe_subscription_id: eventObject.id } },
            );
          }
          break;

        case "customer.subscription.deleted":
          const getData = await db.Subscription.findOne({
            where: { stripe_subscription_id: eventObject.id },
          });

          if (eventObject.status === "canceled") {
            await db.Subscription.update(
              { status: "cancelled" },
              { where: { id: getData.id } },
            );

            await db.User.update(
              { activePlan: false, activePlanId: null },
              { where: { id: getData.user_id } },
            );
          }
          break;

        case "invoice.upcoming":
          if (eventObject.subscription) {
            const getSubscriptionData = await db.Subscription.findOne({
              where: { stripe_subscription_id: eventObject.subscription },
            });

            if (getSubscriptionData) {
              const getSubscriptionPlan = await db.Plan.findByPk(
                getSubscriptionData.subscription_plan_id,
              );
            }
          }
          break;

        case "checkout.session.completed":
          try {
            if (eventObject.metadata.webhookIdentifier === "Subscription") {
              if (eventObject) {
                await db.User.update(
                  {
                    activePlan: true,
                    activePlanId: eventObject.metadata.plan_id,
                  },
                  { where: { id: eventObject.metadata.user_id } },
                );

                const findPlan = await db.Plan.findByPk(
                  eventObject.metadata.plan_id,
                );
                if (!findPlan) {
                  return response.status(404).send("Plan not found");
                }

                const subscriptionObj = {
                  subscription_plan_id: findPlan.id,
                  user_id: eventObject.metadata.user_id,
                  stripe_subscription_id: eventObject.subscription,
                  amount: eventObject.metadata.amount * 100,
                  interval_count: eventObject.metadata.interval_count,
                };

                const getExistingSubscription = await db.Subscription.findOne({
                  where: {
                    user_id: eventObject.metadata.user_id,
                    status: "active",
                  },
                });

                if (getExistingSubscription) {
                  const getStripeExistingSubscription =
                    await stripeServices.retrieve_subscrition({
                      stripe_subscription_id:
                        getExistingSubscription.stripe_subscription_id,
                    });

                  if (getStripeExistingSubscription) {
                    const deleteOldSubscription =
                      await stripeServices.delete_subscription({
                        stripe_subscription_id:
                          getExistingSubscription.stripe_subscription_id,
                      });

                    if (
                      deleteOldSubscription &&
                      deleteOldSubscription.status == "canceled"
                    ) {
                      await db.Subscription.update(
                        {
                          updatedBy: eventObject.metadata.user_id,
                          status: "cancelled",
                        },
                        { where: { id: getExistingSubscription.id } },
                      );
                    }
                  }
                }

                const getStripeSubscription =
                  await stripeServices.retrieve_subscrition({
                    stripe_subscription_id: eventObject.subscription,
                  });

                if (getStripeSubscription) {
                  subscriptionObj.valid_upto = new Date(
                    getStripeSubscription.current_period_end * 1000,
                  );
                }

                subscriptionObj.status = "active";
                await db.Subscription.create(subscriptionObj);

                const transactionPayload = {
                  user_id: eventObject.metadata.user_id,
                  paid_to: findPlan.addedBy,
                  transaction_type: "buy_subscription",
                  paymentMethod: "Stripe",
                  subscription_plan_id: findPlan.id,
                  stripe_subscription_id: eventObject.subscription,
                  payment_intent_id: eventObject.invoice,
                  amount: eventObject.amount_total || 0,
                  transaction_status:
                    eventObject.payment_status === "paid"
                      ? "successful"
                      : "failed",
                };

                await db.Transaction.create(transactionPayload);
              }
            }
          } catch (error) {
            console.log(error);
          }
          break;

        default:
          console.warn(`Unhandled event type: ${event.type}`);
      }

      response.json({ received: true });
    } catch (error) {
      response.status(500).send("Internal Server Error");
    }
  },

  payNowOnStripe: async (req, res) => {
    try {
      let data = req.body;
      let transaction_payload = {};
      let find_user = await db.User.findOne({
        where: {
          id: req.identity.id,
          isDeleted: false,
        },
      });

      if (find_user) {
        var email = find_user.email;
      }

      let find_plan = await db.Plan.findByPk(data.plan_id);

      transaction_payload.user_id = req.identity.id;
      transaction_payload.plan_id = find_plan.id;

      let get_admin = await db.User.findAll({
        where: {
          role: "admin",
          isDeleted: false,
        },
      });

      transaction_payload.paid_to = get_admin[0].id;

      if (find_plan) {
        for (let price_obj of find_plan.pricing) {
          if (
            price_obj.interval == req.body.interval &&
            price_obj.unit_amount == req.body.amount &&
            price_obj.interval_count == req.body.interval_count
          ) {
            var stripe_price = price_obj.stripe_price_id;
          }
        }
      }

      let priceObj = {};
      priceObj.quantity = 1;
      priceObj.price = transaction_payload.amount;
      let line_items = [
        {
          price: stripe_price ? stripe_price : "",
          quantity: 1,
        },
      ];

      if (data.type == "buy") {
        let create_session = await stripeServices.one_time_payment({
          line_items: line_items,
          email: email,
          userId: find_user.id,
          metadata: {
            plan_id: transaction_payload.plan_id,
            user_id: transaction_payload.user_id,
            amount: req.body.amount,
            interval_count: req.body.interval_count,
            webhookIdentifier: "Subscription",
          },
          role: find_user.role,
        });

        if (create_session) {
          let resData = {
            url: create_session.url,
          };
          return res.status(200).json({
            success: true,
            code: 200,
            data: resData,
          });
        }
      } else {
        const cancleCurrentSubscription =
          await stripeServices.delete_subscription({
            stripe_subscription_id: req.body.subscriptionId,
          });

        if (cancleCurrentSubscription) {
          let create_session = await stripeServices.one_time_payment({
            line_items: line_items,
            email: email,
            userId: find_user.id,
            metadata: {
              plan_id: transaction_payload.plan_id,
              user_id: transaction_payload.user_id,
              amount: req.body.amount,
              interval_count: req.body.interval_count,
              webhookIdentifier: "Subscription",
            },
            role: find_user.role,
          });

          if (create_session) {
            let resData = {
              url: create_session.url,
            };
            return res.status(200).json({
              success: true,
              code: 200,
              data: resData,
            });
          } else {
            return res.status(400).json({
              success: false,
              error: {
                code: "400",
                message: constants.SUBSCRIPTION.ERROR_IN_SUBSCRIPTION_UPDATION,
              },
            });
          }
        }
      }
    } catch (error) {
      console.log(error);
      return res.status(400).json({
        success: false,
        error: {
          code: 400,
          message: error.message,
        },
      });
    }
  },

  active_Subscription: async (req, res) => {
    try {
      const userId = req.query.user_id;

      if (!userId) {
        return res.status(400).json({
          success: false,
          message: "User id is required",
        });
      }

      const findUser = await db.User.findOne({
        where: {
          id: userId,
          isDeleted: false,
        },
      });

      if (findUser) {
        const plans = await db.Subscription.findOne({
          where: {
            user_id: userId,
            status: "active",
          },
          include: [
            {
              model: db.Plan,
              as: "subscription_plan",
            },
          ],
          order: [["createdAt", "DESC"]],
        });

        if (plans) {
          const data = plans.get({ plain: true });

          data.subscription_plan.pricing.forEach((priceObj) => {
            priceObj.isActive = priceObj.interval_count === data.interval_count;
          });

          return res.status(200).json({
            success: true,
            data,
          });
        } else {
          return res.status(200).json({
            success: false,
            data: {},
          });
        }
      } else {
        return res.status(404).json({
          success: false,
          error: {
            code: "404",
            message: constants.USER.NOT_FOUND,
          },
        });
      }
    } catch (error) {
      return res.status(400).json({
        success: false,
        message: error.message || "An error occurred",
      });
    }
  },

  cancelSubscription: async (req, res) => {
    try {
      let planId = req.body.plan_id;
      let userId = req.body.user_id;

      // Find the existing active subscription for the user and plan
      let get_existing_subscription = await db.Subscription.findOne({
        where: {
          user_id: userId,
          subscription_plan_id: planId,
          status: "active",
        },
      });

      // If no active subscription is found, return an error
      if (!get_existing_subscription) {
        return res.status(400).json({
          success: false,
          code: 400,
          message: constants.SUBSCRIPTION.NOT_FOUND,
        });
      }

      // Retrieve the subscription from Stripe
      let get_stripe_existing_subscription =
        await stripeServices.retrieve_subscrition({
          stripe_subscription_id:
            get_existing_subscription.stripe_subscription_id,
        });

      // If the Stripe subscription is already canceled, update the database
      if (get_stripe_existing_subscription.status == "canceled") {
        await db.User.update(
          { activePlan: false, activePlanId: null },
          { where: { id: userId } },
        );

        await db.Subscription.update(
          {
            updatedBy: userId,
            status: "cancelled",
          },
          { where: { id: get_existing_subscription.id } },
        );

        return res.status(200).json({
          success: true,
          code: 200,
          message: constants.SUBSCRIPTION.CANCELED,
        });
      }

      // If the Stripe subscription is active, cancel it
      if (get_stripe_existing_subscription) {
        let delete_old_subscription = await stripeServices.delete_subscription({
          stripe_subscription_id:
            get_existing_subscription.stripe_subscription_id,
        });

        // If the subscription is successfully canceled, update the database
        if (
          delete_old_subscription &&
          delete_old_subscription.status == "canceled"
        ) {
          await db.User.update(
            { activePlan: false, activePlanId: null },
            { where: { id: userId } },
          );

          await db.Subscription.update(
            {
              updatedBy: userId,
              status: "cancelled",
            },
            { where: { id: get_existing_subscription.id } },
          );

          return res.status(200).json({
            success: true,
            code: 200,
            message: constants.SUBSCRIPTION.CANCELED,
          });
        } else {
          // If the subscription cancellation fails, return an error
          return res.status(400).json({
            success: false,
            error: {
              code: 400,
              message: constants.SUBSCRIPTION.CANCEL_FAILED,
            },
          });
        }
      } else {
        // If the Stripe subscription is not found, return an error
        return res.status(400).json({
          success: false,
          error: {
            code: 400,
            message: constants.SUBSCRIPTION.NOT_FOUND,
          },
        });
      }
    } catch (error) {
      // Handle any unexpected errors
      return res.status(400).json({
        success: false,
        message:
          error.message || "An error occurred while canceling the subscription",
      });
    }
  },
  getAllSubscriptions: async (req, res) => {
    try {
        const { count, page, search, status, userId, sortBy } = req.query;

        // Base query options
        const queryOptions = {
            include: [
                {
                    model: db.User,
                    as: 'userDetail',
                    attributes: ['id', 'fullName'], // Include only necessary fields
                    required: false, // Use LEFT JOIN
                },
                {
                    model: db.Plan,
                    as: 'planDetails',
                    attributes: ['id', 'name'], // Include only necessary fields
                    required: false, // Use LEFT JOIN
                },
            ],
            where: {}, // Initialize empty where clause
            order: [['createdAt', 'DESC']], // Default sorting
        };

        // Add search filter
        if (search) {
            queryOptions.where[Op.or] = [
                { '$userDetail.fullName$': { [Op.like]: `%${search}%` } },
                { '$planDetails.name$': { [Op.like]: `%${search}%` } },
            ];
        }

        // Add status filter
        if (status) {
            queryOptions.where.status = status;
        }

        // Add user ID filter
        if (userId) {
            queryOptions.where.user_id = userId;
        }

        // Add sorting
        if (sortBy) {
            const [field, sortType] = sortBy.split(" ");
            queryOptions.order = [[field || 'createdAt', sortType === 'desc' ? 'DESC' : 'ASC']];
        }

        // Count total records (for pagination)
        const total = await db.Subscription.count({
            where: queryOptions.where,
            include: queryOptions.include,
        });

        // Add pagination
        if (page && count) {
            const offset = (Number(page) - 1) * Number(count);
            queryOptions.offset = offset;
            queryOptions.limit = Number(count);
        }

        // Fetch subscriptions with filters and pagination
        const result = await db.Subscription.findAll(queryOptions);

        return res.status(200).json({
            success: true,
            data: result,
            total: total,
        });
    } catch (err) {
        console.error("Error in getAllSubscriptions:", err);
        return res.status(500).json({
            success: false,
            error: { code: 500, message: err.message || "An error occurred" },
        });
    }
},

};
```




<!--- Eraser file: https://app.eraser.io/workspace/McqPk0eq3RJ7X81a1d8F --->